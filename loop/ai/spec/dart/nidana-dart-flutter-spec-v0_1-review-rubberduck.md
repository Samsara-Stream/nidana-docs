# Nidana Bus Reference Architecture v0.10 — Dart/Flutter Spec Review

**Reviewed file:** `loop/ai/spec/dart/nidana-dart-flutter-spec-v0_1-copilotcli.md`  
**Against:** `arch/nidana-bus-ref-arch.md` (v0.10.2)  
**Method:** Three independent rubber-duck review passes  
**Date:** 2026-04-09

---

## Summary

Three independent rubber-duck passes were run against the Dart/Flutter spec and the reference architecture. The passes surfaced **27 distinct findings** across four severity levels. The spec demonstrates solid structural awareness of the reference architecture but has several blocking issues that make it **not yet implementation-ready**. The most critical problems are in the navigation runtime contract, `StateTopic` read semantics, envelope propagation mechanics, and topology identity.

| Severity | Count |
|---|---|
| 🔴 Critical (blocks correct runtime behaviour) | 7 |
| 🟠 Significant (design flaw or missing required contract) | 13 |
| 🟡 Minor (notable omission, weaker than ref arch) | 7 |

---

## 🔴 Critical

### C-1 — `Topology` has no required `name` field

**Spec location:** §4.1 (lines 324–330)  
**Ref arch:** §2.3, §10.2, §14.2

The `Topology` abstract class exposes no `name`. The reference architecture requires topology identity for: envelope `source` attribution (`"topology:checkout-flow"`), graph node labels, cycle detection, error reporting, and the devtools catalog. `TopologyHandle` carries a `topologyName` but that field has no authoritative origin without a `name` on the topology itself.

**Fix:** Add `String get name;` (or a required constructor argument) to `Topology`. All bus/devtools code must derive topology identity from this field.

---

### C-2 — `getCurrentValue()` throws on first reference; ref arch requires initialize-and-return

**Spec location:** §3.3 (lines 197–199, 218)  
**Ref arch:** §3.3, §2.5

The spec states `getCurrentValue(topic)` throws `StateError` if the topic "has not been initialized." The ref arch's invariant is: `StateTopic` is **always readable** — the bus creates and seeds the backing `BehaviorSubject` on first reference. Both `TopicBuilder` and `TopicBloc` call `getCurrentValue()` as their first operation, so first-paint UI views will throw.

**Fix:** Define `getCurrentValue(StateTopic<T>)` as a first-reference initializer: create the subject, seed it with `topic.initial()`, and return the value. It must never throw.

---

### C-3 — `NavigationCoreTopology` uses `combineLatest2`, causing stale intent replay

**Spec location:** §5.3 (lines 550–558)  
**Ref arch:** §8.3

```dart
// ❌ As specified
combineLatest2(intents, auth, (intent, state) => ...)
```

`NavIntent` is an `EventTopic` (cold per emission). `AuthState` is a `StateTopic` (always has a value). `combineLatest2` re-emits the most recent intent on **every auth state change**, causing spurious duplicate navigation whenever the user logs in/out.

**Fix:** Use `withLatestFrom`-style semantics — emit only when a **new intent** arrives, using the latest auth state as context:

```dart
// ✅ Correct
intents.withLatestFrom(auth, (intent, state) => ...)
```

---

### C-4 — `NavigationExecutor` is never subscribed to `resolvedIntent`

**Spec location:** §5.4 (lines 581–583), §6.6 (lines 695–723)  
**Ref arch:** §8.3

The spec defines `NavigationCoreTopology` producing `resolvedIntent`, and defines `NavigationExecutor` with an `execute()` method — but no component is specified to subscribe to `resolvedIntent` and call `execute()`. Navigation never fires.

**Fix:** Specify an app-eager navigation service (e.g. `NavigationBinder`) that activates alongside the bus, subscribes to `NavigationTopics.resolvedIntent`, and delegates each emission to the executor.

---

### C-5 — `NavigationTopics.currentRoute` has no producer

**Spec location:** §5.2 (lines 523–527), §5.3 (lines 557–558), §6.6 (lines 701–721)  
**Ref arch:** §8.3

`currentRoute` is declared as a core output `StateTopic` but nothing in the spec writes to it. Downstream consumers (history derivation, back-stack logic, module lifecycle) read a topic that is always `RouteState.initial()`.

**Fix:** Specify that either the `NavigationBinder` (post-execution) or a `GoRouter` redirect/listener publishes confirmed route transitions to `currentRoute` after successful navigation.

---

### C-6 — `_toRouteTransition` is referenced but not defined

**Spec location:** §5.3 (line 558)  
**Ref arch:** §8.3

`history` is derived from `resolvedIntent.map(_toRouteTransition)`, but `_toRouteTransition` is never defined anywhere in the spec. This is a compile error in any skeleton implementation generated from this spec.

**Fix:** Define `_toRouteTransition` explicitly, noting that it requires both `previous currentRoute` and the `resolvedIntent` to construct a `RouteTransition(from, to)`. Derive it from `(previousCurrentRoute, resolvedIntent)` after execution confirmation, not from the intent stream alone.

---

### C-7 — `nidana_dart_navigation` hard-codes `AuthTopics` / `AuthState`

**Spec location:** §5.3 (lines 551–572), package dependency graph §2.2–§2.3  
**Ref arch:** Appendix C.3, §8.3

The package is declared as a generic, reusable library, yet its core topology directly imports and uses `AuthTopics` and `AuthState`. This makes the package non-generic and introduces an undeclared dependency on the app's auth domain shape.

**Fix:** Move auth-specific guard logic to app code. Make `NavigationCoreTopology` accept generic guard streams/context, composable by the host application.

---

## 🟠 Significant

### S-1 — Envelope causation/correlation propagation has no specified mechanism

**Spec location:** §3.2 (line 175), §3.3 (lines 188–196), §4.2 (lines 344–350)  
**Ref arch:** §2.4, §5.2, §10.1

`read()` returns `Stream<T>` and `write()` accepts `Stream<T>`, stripping envelope metadata entirely from the topology body. The bus has no specified way to thread `causationId` / `correlationId` through a topology activation. This makes the core traceability contract undefined at the runtime level.

**Fix:** Specify the internal mechanism explicitly — e.g. zone-local active-envelope context, envelope-carrying internal streams, or context-preserving wrappers — and define how `b.write()` captures the parent envelope from the current activation context.

---

### S-2 — `applicationLazy` lifecycle scope is unimplementable as specified

**Spec location:** §6.1 (line 599), §6.2 (lines 609–626)  
**Ref arch:** §6.1–§6.4

"Activate on first read/write to a handled topic" requires a bus-level registry mapping topics → lazy topologies, checked before the first access to any topic. No such registration or lookup mechanism is specified in the bus or topology APIs.

**Fix:** Require that topology `declare()` captures its reads/writes at registration time, and that the bus maintains a lazy-topology registry. Define the lookup path: `bus.read(topic)` → check lazy registry → activate if needed → return stream.

---

### S-3 — `TopologyBuilder.on()` is incoherent as specified

**Spec location:** §4.2 (lines 351–353)  
**Ref arch:** §5.1, §10.2

`on()` accepts a `void Function(T value)` handler and the comment says it should "publish results via the builder." The builder's `write()` only accepts streams; there is no declarative way to emit from an imperative callback. This pushes topology authors toward `bus.publish(...)` side-effects that bypass graph visibility and violate the ref arch's topology-as-data model.

**Fix:** Remove `on()` from the public API, or redesign it as a declarative operator with explicit named output streams (e.g. `on<T, R>(topic, transform)` that returns a `Stream<R>` wired into a named output).

---

### S-4 — `StateTopic` pure factory `() => T` is a ref-arch requirement, deferred as open question

**Spec location:** §3.1 (lines 117–124, 146), §14 (line 972)  
**Ref arch:** §3.3

The ref arch explicitly supports `() => T` for runtime-constructed defaults (e.g. `Uuid.v4()`, system clock). The spec's concrete API only accepts `initial: T` (a constant), deferring factory support to an open question.

**Fix:** Include `factory: () => T` as part of the baseline `StateTopic` API alongside `initial: T`. This is not a question to defer — it's a specified requirement.

---

### S-5 — Topic uniqueness enforcement ignores variant and config mismatch

**Spec location:** §3.1 (lines 148–149), §3.3 (line 217)  
**Ref arch:** §3.3, §2.5

The spec rejects name + `T` conflicts at runtime, but silently accepts: same name as `StateTopic<T>` vs `EventTopic<T>` (same type, different variant), or same name as `ReplayTopic<T>` with different buffer sizes. These produce data integrity bugs and non-deterministic initial-value behaviour.

**Fix:** Uniqueness enforcement must cover `(name, variant, config)` — not just `(name, T)`.

---

### S-6 — `toGraph()` introspection mechanism is unspecified

**Spec location:** §4.3 (lines 385–414)  
**Ref arch:** §5.3, §17.1

`TopologyIntrospection.toGraph()` promises graph extraction without activating the topology, but the spec never defines *how*. The mechanism — that `declare()` is called against an inert recording builder that captures `read()`/`write()` calls as metadata rather than wiring real subjects — is left implicit.

**Fix:** State explicitly that `declare()` runs in two modes: live (real subjects) and recording (metadata capture). Define the `RecordingTopologyBuilder` contract used for `toGraph()`, `systemGraph()`, and the codegen catalog.

---

### S-7 — `NidanaBus.instance` singleton blocks TestBus injection for widget tests

**Spec location:** §3.3 (line 220), §6.5, §7.1  
**Ref arch:** §17.1 (testing levels)

`TopicBuilder` and `TopicBloc` hardcode `NidanaBus.instance`. `NidanaApp` has no `bus:` injection parameter. Level 4 widget tests require "inject a TestBus," which is impossible without a bus resolution mechanism in the widget tree.

**Fix:** Add `bus:` parameter to `NidanaApp`. Resolve bus via `NidanaScope.of(context)` in `TopicBuilder` / `TopicBloc`. Fall back to `NidanaBus.instance` only when no scope is present.

---

### S-8 — Module-scope lifecycle binding is underspecified for GoRouter

**Spec location:** §6.1, §6.4  
**Ref arch:** §6.1–§6.4, §14.2

The spec says module scope is driven by `RouteObserver` / route-aware integration and also standardises on GoRouter. These are not reconciled. No mechanism is defined for keeping module-scoped topologies alive across intra-module route changes and tearing them down on module exit.

**Fix:** Define a concrete `GoRouter`-compatible module scope binding: a `ShellRoute` shell widget that activates the module topology bundle on entry and deactivates it on exit, using GoRouter's route lifecycle callbacks.

---

### S-9 — Route registry pattern is absent

**Spec location:** §5.1 (lines 492–495)  
**Ref arch:** §8.3

The ref arch treats `Route` constants like `Topic` constants: typed, named, organized in registries for discoverability and governance. The spec defines `Route` as a value object but omits registries entirely, losing the ref arch's typed-navigation discipline.

**Fix:** Introduce a `RouteRegistry` pattern mirroring `TopicRegistry`. Routes are static final constants; the registry is the single source of truth for route definitions used by guards, executors, and tests.

---

### S-10 — Navigation result handling is missing

**Spec location:** §5.1 (lines 492–495), §5.2, §6.6 (line 720)  
**Ref arch:** §8.3

`BackWithResult` is defined as a nav intent variant, but the result publication pattern — module-scoped result topic, correlation-based linking of original intent to result, cleanup policy — is not specified anywhere.

**Fix:** Document the result-topic pattern: a module-scoped `EventTopic<R>` published by the child route on back, linked to the originating intent via `correlationId`, and cleaned up when the module scope is torn down.

---

### S-11 — Topic-vs-topology lifecycle semantics are not documented

**Spec location:** §6.1–§6.4  
**Ref arch:** §6.5

The ref arch's §6.5 explicitly documents that `StateTopic` values persist across topology deactivation/reactivation (topics outlive their topologies). This is a central architectural property that this spec leaves implicit.

**Fix:** Add a dedicated sub-section mirroring ref §6.5, explicitly stating topic-vs-topology ownership and persistence semantics.

---

### S-12 — Topic cleanup policy is deferred as open question

**Spec location:** §3.3 (line 208), §14 (line 976)  
**Ref arch:** §6.5, §3.3

Memory and retention behaviour for topics is left undefined despite the ref arch recommending a hybrid default: `StateTopic` permanent, `EventTopic` auto-GC after all subscribers drop, `ReplayTopic` configurable.

**Fix:** Specify the default cleanup policy now. Make it an explicit design decision, not an open question.

---

### S-13 — Error handling coverage is incomplete

**Spec location:** §11.3 (lines 917–922)  
**Ref arch:** §9

"Transformers must not throw" is insufficient. The ref arch also requires: stream-level `catchError` / `onErrorResumeNext`-style containment on every topology output stream, a global `Topic<AppError>` publication path for fatal topology errors, and clear semantics for whether a topology is restarted or deactivated on stream termination.

**Fix:** Add stream-level error containment conventions and define how topology-level fatal errors propagate to the global error topic.

---

## 🟡 Minor

### M-1 — `BackTo` and `DeepLink` executor semantics are wrong

**Spec location:** §6.6  
**Ref arch:** §8.3

`BackTo` is implemented as `router.go(...)` — a push/replace, not pop-until semantics. `DeepLink` uses `uri.path` only, silently dropping query parameters and fragment.

**Fix:** `BackTo` → `router.popUntil(...)` or equivalent stack-inspection approach. `DeepLink` → parse full `Uri` including query/fragment.

---

### M-2 — Approach C (reference-counted scopes) is omitted

**Spec location:** lifecycle sections §6.1–§6.4  
**Ref arch:** §6.3

The ref arch describes Approach C (reference-counted scope with grace period) as a real option for rapid page transitions, where the topology is not torn down until all references drop plus a grace period. The spec omits it entirely with no mention.

**Fix:** Mention Approach C as deferred/optional; note its intended use case for high-frequency navigation.

---

### M-3 — Audit topic approach is not documented

**Spec location:** §3.4  
**Ref arch:** §10.1

The ref arch offers two audit approaches: interceptors (Approach A) and a dedicated `Topic<AuditEntry>` (Approach B). The spec only covers interceptors, losing Approach B which is preferred for devtools integration.

**Fix:** Document `Topic<AuditEntry>` as optional Approach B, note it as the preferred path for `nidana_dart_devtools`.

---

### M-4 — Scope-violation detection is absent

**Spec location:** (no equivalent section)  
**Ref arch:** §11.4

The ref arch requires runtime dev-mode warnings when a page-scoped topology writes to an app-scoped topic. No equivalent mechanism is specified in the Dart spec.

**Fix:** Add runtime dev-mode assertions checked on `b.write()` comparing topology scope to topic scope, with a CI validation path using the topology catalog.

---

### M-5 — Protobuf upgrade path is missing

**Spec location:** §3.6, §11.2  
**Ref arch:** §3.6

The ref arch explicitly calls out protobuf as the upgrade path for cross-platform contract sharing and persistence compatibility. The spec only covers native Dart classes and `freezed`.

**Fix:** Add a note on when/why to upgrade from native contracts to protobuf (cross-platform sharing, wire compatibility, schema evolution).

---

### M-6 — `systemGraph()` behaviour around lazy topologies is unspecified

**Spec location:** §4.4 (line 439), §6.1 (line 599)  
**Ref arch:** §17.1

If lazy topologies are invisible in `systemGraph()` until first activation, observability output will be misleading. The spec does not define whether `systemGraph()` reports active-only or registered+active topology status.

**Fix:** Define `systemGraph()` as reporting all registered topologies with an `active` / `registered-lazy` status flag.

---

### M-7 — Ownership/governance annotations are YAML-only

**Spec location:** §10.2 (codegen schema), manual registry sections  
**Ref arch:** §11.4, §17.1

`owner` metadata exists in the YAML codegen schema but is absent from hand-written topic registries. Governance enforcement is silently skipped on the non-codegen path.

**Fix:** Add optional `owner` annotation to the topic declaration API; require CI scanning of all registries (codegen and hand-written) for ownership completeness.

---

## Recommended Fix Priority

Before this spec is used to drive package implementation:

1. **C-1** — Add `name` to `Topology`
2. **C-2** — Fix `getCurrentValue` to initialize-and-return
3. **C-3** — Fix `combineLatest2` → `withLatestFrom`
4. **C-4** — Wire `NavigationBinder` to subscribe `resolvedIntent`
5. **C-5** — Specify `currentRoute` producer
6. **C-6** — Define `_toRouteTransition`
7. **C-7** — Remove `AuthTopics` coupling from navigation package
8. **S-1** — Specify envelope propagation mechanism
9. **S-2** — Specify lazy activation registry
10. **S-6** — Define `RecordingTopologyBuilder` for `toGraph()`
11. **S-7** — Add `bus:` injection to `NidanaApp`
12. **S-4** — Add `factory: () => T` to `StateTopic` baseline API
