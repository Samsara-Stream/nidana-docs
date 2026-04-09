# Nidana Bus Reference Architecture v0.10 — Self-Review

**Reviewed document:** `arch/nidana-bus-ref-arch-v0_10.md` (v0.10.2)  
**Method:** Three independent rubber-duck review passes  
**Passes:** Core runtime & lifecycle · Platform implementability · Navigation, error handling, testing & DevTools  
**Date:** 2026-04-09

---

## Summary

Three independent rubber-duck passes reviewed the reference architecture for internal gaps, inconsistencies, and implementability across all four target platforms (Dart/Flutter, Kotlin/Android, Swift/iOS, TypeScript/Web). The architecture is coherent at the conceptual level and its core property claims (typed topics, declarative topologies, pure-core/imperative-shell separation) are sound. However, several implementation-blocking gaps exist — particularly in envelope propagation semantics for multi-input operators, `on()` handler design, lazy scope activation, subject creation timing, and the navigation pipeline contract.

| Severity | Count |
|---|---|
| 🔴 Critical (blocks correct implementation) | 16 |
| 🟠 Significant (design gap or platform-specific blocker) | 27 |
| 🟡 Minor (clarification needed, low risk) | 10 |

---

## 🔴 Critical

### C-1 — Envelope causation lineage for multi-input operators is undefined

**Section:** §2.4  
**Platforms:** All

§2.4 defines envelope threading for the single-input case: "the bus attaches the incoming message's `id` as the outgoing message's `causationId`." But `combine`/`combineLatest`/`withLatestFrom` have **N input envelopes arriving at the same operator**. The spec does not define:

- Which input envelope becomes the `causationId` of the output message
- What happens when N inputs fire simultaneously
- What happens when inputs carry different `correlationId`s (cross-flow combine)
- Whether a multi-parent causal structure requires `parentIds[]` instead of a single `causationId`

Different runtimes will make incompatible choices, silently breaking traceability across platforms.

**Fix:** Add a normative "envelope lineage rules for combinators" sub-section to §2.4. At minimum: define the "trigger" input (the one whose emission caused the output), declare its envelope as the causation parent, define `correlationId` merge rules (keep, assert-equal, or thread the dominant one), and state whether degenerate cases (simultaneous triggers) require platform documentation.

---

### C-2 — `on()` handler is inconsistent with the pure-wiring model and breaks graph extraction

**Section:** §5.1, §5.5, §11.4  
**Platforms:** All

`on(topic) { value -> write(otherTopic, value) }` is imperative — the handler receives a scalar `T` and calls `write()`. But `write()` in the builder accepts a `Stream<T>`, and the topology body is declared to be "pure inert wiring." The inconsistency has two consequences:

1. `on()` handlers can publish to **undeclared topics** dynamically (explicitly acknowledged in §10.2), which makes `toGraph()`, cycle detection, and orphan analysis **unsound**.
2. The handler semantics are never defined: is it `read(topic).flatMap(...) -> write(...)`? Can it write to multiple topics? Can it be conditional?

**Fix:** Either (a) remove `on()` from the public DSL and show the equivalent as `read(topic).switchMap(...)` / `flatMap(...)` piped into `write()`, or (b) define `on()` strictly as syntactic sugar over `read + map/flatMap + write` with all outputs declared statically at declaration time. Either way, prohibit dynamic writes to undeclared topics.

---

### C-3 — `applicationLazy` scope: bus cannot know handled topics before activation

**Section:** §6.1, §6.2  
**Platforms:** All

"Activate on first interaction" requires the bus to maintain a registry from topic references → lazy topologies, checked before any access to those topics. But:

1. The spec never defines how the bus learns which topics a lazy topology handles **before** `declare()` is called.
2. If the first interaction is a **publish to an EventTopic**, the event may be lost: the topology is activated *after* the publish, so the event fires into a cold subscription.
3. §5.1 prohibits "reading a topic's current value synchronously to decide which streams to wire," but lazy activation requires exactly that kind of pre-activation topic inspection.

**Fix:** Require that topology definitions provide a pre-activation metadata struct (topic handle sets for reads and writes) — separate from `declare()`. Define the lazy activation contract: `activate_then_deliver` for EventTopic triggers (queue the message, activate the topology, replay the message). Specify the registration API: `bus.registerLazy(topology)`.

---

### C-4 — Subject creation timing: three contradictory statements

**Section:** §2.5, §5.5, §14.4  
**Platforms:** All

The document makes three incompatible claims about when the bus creates a backing reactive subject:

1. §2.5: "The bus creates the subject on first reference **via a `read()` or `write()` call in any topology declaration**"
2. §14.4 comment: "Subject is created here, **on first reference during topology activation**"
3. §5.5 / §11.4: topology declarations are **inert data structures** introspectable without activation

These cannot all be true. If `declare()` creates subjects, declarations are not inert. If subjects are created on activation, the §2.5 wording is wrong. If declarations are inert, what does "first reference" mean?

**Fix:** Adopt one model and state it clearly. The cleanest: **declaration never creates subjects**; subject creation happens on the first runtime access (`activate`, `publish`, `observe`, or `getCurrentValue`). `declare()` records read/write metadata into a `TopologyDefinition` IR. The bus interprets this IR on activation.

---

### C-5 — `getCurrentValue()` on an unobserved StateTopic is never defined

**Section:** §2.5, §15.2  
**Platforms:** React (explicitly), all others implicitly

§15.2 shows `useSyncExternalStore(() => bus.getCurrentValue(topic))` called **before any topology is activated**, relying on the StateTopic being readable. §2.5 says the bus creates subjects on first reference. But the spec never states:

- Whether `getCurrentValue()` is a "first reference" that materializes the subject
- Whether it returns `initial` before any publish has occurred
- Whether it throws if no subject exists

The architecture's invariant "StateTopic is always readable" is implied by §2.5 but never made a normative rule.

**Fix:** Add to §2.5: "`getCurrentValue(StateTopic)` is always defined. If no backing subject exists, the bus creates it, evaluates `initial` (or calls the factory once), and returns the result. It never throws."

---

### C-6 — `NavigationTopics.resolvedIntent` is missing from the topic registry

**Section:** §8.3  
**Platforms:** All

The navigation topology pseudocode writes to `NavigationTopics.resolvedIntent`, and the `NavigationExecutor` must subscribe to it. But `NavigationTopics` as defined contains only `intent`, `currentRoute`, and `history`. `resolvedIntent` is never declared.

**Fix:** Add `resolvedIntent: EventTopic<NavIntent>` (or a post-resolution ADT `ResolvedNavIntent`) to `NavigationTopics`. Define that `NavigationExecutor` subscribes to this topic on activation. State that `resolvedIntent` must not contain `DeepLink` variants (they are resolved before writing here).

---

### C-7 — `intent.requiresAuth` property does not exist on `NavIntent`

**Section:** §8.3  
**Platforms:** All

`applyAuthGuard` calls `intent.requiresAuth`, but the `NavIntent` ADT has no such property. This is unimplementable as written.

**Fix:** Define the auth policy source. The recommended model: a `RouteAccessPolicy` map keyed by `Route` (analogous to the role-based RBAC example in the same section). Guards inspect the target route, not the intent. Example: `val policy = routeAccessPolicy[intent.route] ?: return intent`.

---

### C-8 — Navigation guard uses the wrong combinator; causes stale intent replay

**Section:** §8.3  
**Platforms:** All

```
val guarded = combine(intents, auth, ::applyAuthGuard)
```

`NavigationTopics.intent` is an `EventTopic` (no replay). `AuthTopics.state` is a `StateTopic` (always has a current value). `combineLatest`/`combine` will **re-emit the most recent nav intent on every auth state change** — causing spurious duplicate or stale navigation whenever the user logs in or out.

The correct operator is `withLatestFrom`: emit only when a **new intent arrives**, using the latest auth state as context.

**Fix:** Replace with `intents.withLatestFrom(auth, ::applyAuthGuard)`. If auth changes should proactively trigger redirects (e.g. force to login on token expiry), that is a **separate** topology operating on `currentRoute × auth`, not the intent processing pipeline.

---

### C-9 — `toRouteTransition` is undefined; requires state not available in the intent stream

**Section:** §8.3  
**Platforms:** All

```
write(NavigationTopics.history, resolved.map(::toRouteTransition))
```

`toRouteTransition` is called but never defined. A `RouteTransition(from, to)` requires the **previous route** — but the `resolved` stream only carries the current `NavIntent`. The previous route is not accessible inside the navigation topology.

**Fix:** Define `RouteTransition` derivation from **confirmed route state changes**: `pairwise(currentRoute)` after the executor confirms navigation. History must derive from router-observed transitions, not intent emission. Alternatively, use `zipWith(currentRoute)` to capture the "from" state at intent time — but document the timing implications.

---

### C-10 — `currentRoute` has no producer; the topic is always `RouteState.initial()`

**Section:** §8.3  
**Platforms:** All

`NavigationTopics.currentRoute` is declared as a `StateTopic<RouteState>` but **nothing in the spec writes to it**. The navigation topology writes to `resolvedIntent` and `history`, not `currentRoute`. The `NavigationExecutor` performs navigation but is not shown updating `currentRoute`.

**Fix:** Explicitly name the sole writer. The natural model: the platform-specific `NavigationExecutor` implementation publishes to `currentRoute` via a router callback/listener after the platform confirms the route transition. Define this as a contract on `NavigationExecutor` implementations.

---

### C-11 — History and `currentRoute` are derived from intent, not confirmed navigation

**Section:** §8.3  
**Platforms:** All

With the current (broken) design, both `history` and `currentRoute` would be updated at **intent publication time**, before the router has actually performed the navigation. A navigation that fails (permission denied by router, route not found, guard redirect) would still appear as a successful transition in `history` and `currentRoute`.

**Fix:** Define two distinct stages: **requested navigation** (intent → resolvedIntent) and **confirmed navigation** (executor completes → currentRoute updated). `history` and `currentRoute` must derive from confirmed transitions only. The executor or a router observer is the sole writer.

---

### C-12 — Kotlin `StateFlow` suppresses equal values; contradicts §2.3

**Section:** §2.3, §14.1  
**Platforms:** Kotlin/Android

§2.3 explicitly states: "`distinctUntilChanged` is not implicit. Topologies that need it should apply it explicitly." But `MutableStateFlow` — listed as the `StateTopic` backing in §14.1 — **suppresses duplicate values by default** (uses `equals()` to compare, drops equal publishes silently). This means the same value published twice produces one emission on Kotlin but two on Dart (RxDart `BehaviorSubject`) and TypeScript (RxJS `BehaviorSubject`).

**Fix:** Either (a) replace `MutableStateFlow` with a `MutableSharedFlow(replay=1, onBufferOverflow=DROP_OLDEST)` wrapper that does not suppress duplicates, or (b) document this as an intentional platform difference and add a migration note for topology authors who rely on all-publishes-seen semantics.

---

### C-13 — SwiftUI `AnyCancellable` cannot live in a `View` struct

**Section:** §14.2  
**Platforms:** Swift/iOS

The lifecycle integration table maps page scope to `.onAppear/.onDisappear`. SwiftUI views are **structs** (value types). `Set<AnyCancellable>` must be a stored property of a **class** to prevent premature deallocation. A cancellable stored in a struct-based `View` is deallocated on the next render.

**Fix:** State explicitly: cancellable storage for page-scoped topology disposal must live in a reference type — an `ObservableObject`/`@StateObject` view model or a controller retained by `@StateObject`. The page scope activation hooks (`.onAppear/.onDisappear` or `.task`) must reference this retained owner.

---

### C-14 — React StrictMode double-invocation causes activation/deactivation cycle

**Section:** §15.2  
**Platforms:** TypeScript/React

React StrictMode (default in development) double-invokes `useEffect`: mount → cleanup → mount. This means `bus.activate(topology)` → `bus.deactivate(handle)` → `bus.activate(topology)` on every component mount. The spec does not address this. Naive implementations either leak a dangling activation or lose topic state between the two activations.

**Fix:** Require that `activate`/`deactivate` for the same topology definition be **idempotent or reference-counted** per coordination domain. Require that topology definitions passed to `useTopology` are **stable references** (declared as module-level constants or wrapped in `useMemo`). Add a dev-mode warning if `useTopology` receives a new object identity on re-render.

---

### C-15 — React application-lazy scope has no registry mechanism

**Section:** §15.2  
**Platforms:** TypeScript/React

The lifecycle table maps `applicationLazy` to "Activated on first `useTopic` call for a topic the topology handles." But `useTopic` only knows the topic it is subscribing to — it has no knowledge of which lazy topology handles that topic. There is no specified registry from `topic → lazy topologies` for the React integration.

**Fix:** Define the registration API: `bus.registerLazy(topology)` at app startup (inside `NidanaProvider`). On first `observe`/`publish`/`getCurrentValue` for any topic in the lazy topology's declared reads/writes, the bus activates the topology before returning. (This is the same fix as C-3 above, applied specifically to the React integration code.)

---

### C-16 — Vue `useTopic` subscribes in `onMounted`, missing emissions between setup and mount

**Section:** §15.4  
**Platforms:** TypeScript/Vue

```typescript
export function useTopic<T>(topic: StateTopic<T>): Readonly<Ref<T>> {
  const state = ref<T>(bus.getCurrentValue(topic)) as Ref<T>;
  onMounted(() => {
    subscription = bus.observe(topic).subscribe(value => { state.value = value; });
  });
  ...
}
```

The composable reads the initial value synchronously in `setup()`, but subscribes to the observable only in `onMounted()`. Any emission between component creation and mount is **silently missed** — the `ref` holds a stale snapshot.

**Fix:** Move the subscription to `setup()` (Composition API supports this). The disposable subscription still completes in `onUnmounted()`. For SSR compatibility, guard the subscription behind `if (typeof window !== 'undefined')` or use Vue's `watchEffect`-based pattern.

---

## 🟠 Significant

### S-1 — Topology name/identity: class-based syntax has no `name` field

**Section:** §2.3, §5.5, §11.4, §14.4  
**Platforms:** All

The pseudocode topology DSL uses `topology("checkout-flow") { ... }` with an explicit name string. But the class-based platform syntax in §14.4 (`class CheckoutTopology extends Topology`) shows no `name` field or constructor argument. Topology identity is required for: envelope `source` attribution, `TopologyGraph` nodes, cycle detection messages, `systemGraph()` output, and devtools display.

**Fix:** Require a stable `topologyId`/`name` in the class-based contract across all platforms. Add it as a required abstract property or constructor parameter to the `Topology` base class/protocol/interface in each platform syntax example.

---

### S-2 — Reentrancy behavior varies by platform without normative guidance

**Section:** §2.3  
**Platforms:** All

§2.3 documents that reentrant publish semantics differ: synchronous for RxDart `BehaviorSubject`, microtask-deferred for Kotlin `StateFlow`, synchronous for RxJS `BehaviorSubject`. For a "platform-agnostic" architecture, this means a topology that produces correct behavior on one platform may deadlock, loop, or skip events on another.

**Fix:** Either (a) prohibit reentrant publish patterns in topology code (add to the "what the body may not do" list in §5.1), or (b) define a minimum normalized contract: all platforms must defer reentrant publishes to the next microtask/tick, with mandatory dev-mode recursion detection.

---

### S-3 — Bus-instance acquisition is ambiguous: singleton vs injection

**Section:** §2.3, §15.2, §17.1  
**Platforms:** All

The spec says "one bus per coordination domain" and mentions `TestBus` injection, but code examples use ambient singleton-style access (`bus.publish(...)`, `NidanaBus.instance`). There is no defined `BusFactory`, `BusProvider`, or context-injection contract. React gets `NidanaProvider`; Flutter, Kotlin, and Swift have no equivalent.

**Fix:** Define bus acquisition explicitly across all platforms: a default singleton binding via DI for production, plus a provider/scope mechanism for tests and SSR. Add Flutter (`InheritedWidget`/provider), Kotlin (`CompositionLocal`/DI binding), and SwiftUI (`@Environment`/`@StateObject`) equivalents to the React `NidanaProvider` pattern.

---

### S-4 — `AppError` contract is referenced but never defined

**Section:** §8.1, §9.3, Appendix B  
**Platforms:** All

`Topic<AppError>` appears in the system diagram, §8.1, and §9.3 as the cross-feature error routing channel. But `AppError` is never defined: no fields, no severity levels, no recoverable/fatal distinction, no `correlationId` link back to the originating operation.

**Fix:** Define a minimal normative `AppError` contract: `kind: ErrorKind`, `severity: Severity`, `message: String`, `correlationId: String`, `source: String`, `recoverable: Boolean`, `cause: Throwable?` (platform-specific). Add `ErrorTopics.appError: StateTopic<AppError?>` or `EventTopic<AppError>` to the cross-cutting concerns section.

---

### S-5 — Virtual scheduler seam: promised but API never defined

**Section:** §2.3, §17.1  
**Platforms:** All

§2.3 says "the bus provides a seam for injecting a scheduler (used in tests to replace wall-clock time with a virtual time scheduler)." §17.1 lists a virtual scheduler as a `nidana-test` deliverable. But the seam API is never defined anywhere. Without it, `debounce`, `throttle`, `delay`, and `retryWhen` operators are non-deterministic in tests.

**Fix:** Define the scheduler injection API: `BusConfig { schedulerProvider: SchedulerProvider }` injected at bus construction. Define `VirtualScheduler` with `advanceTime(ms)`, `tick()`, and `drain()` methods. Show usage in a testing example.

---

### S-6 — `TestBus` behavioral contract is never specified

**Section:** §17.1, Appendix C.3  
**Platforms:** All

`nidana-test` is listed in Appendix C with deliverables "TestBus, assertion helpers, virtual scheduler." §17.1 describes the need in prose. But the `TestBus` API — whether it is a subclass of the production bus, what assertion helpers look like, whether it records all envelopes, how parallel test isolation works — is entirely unspecified.

**Fix:** Define the `TestBus` contract: (a) implements the production `NidanaBus` interface, (b) takes a `VirtualScheduler` at construction, (c) records all envelopes in order, (d) exposes `assertEmitted(topic, matcher)` / `assertNotEmitted(topic)` helpers, (e) is per-instance isolated (no global state). Add a Level 2 and Level 4 test example for each platform.

---

### S-7 — `toGraph()` dry-run mechanism is implied but never specified

**Section:** §5.5, §11.4, Appendix C.3  
**Platforms:** All

§5.5 promises "topology declarations are introspectable data structures" and `toGraph()` produces a `TopologyGraph` IR. §11.4 says graph extraction happens at build time without activation. But `declare()` is a method that wires live reactive subscriptions. How does `toGraph()` extract metadata without creating subjects and wiring streams?

The fix requires topology declarations to be **IR, not executable code**. The document implies this but never specifies the `TopologyDefinition` structure.

**Fix:** Define `TopologyDefinition` as a value type containing: `topologyId`, `scope`, `reads: Set<TopicRef>`, `writes: Set<TopicRef>`, `transforms: List<TransformDescriptor>`. Define a `RecordingTopologyBuilder` that captures `read()`/`write()` calls as `TopicRef` entries without creating subjects. `declare()` runs against this recorder; the IR is stored and used for both `toGraph()` and live activation. Add this to §5.5 and Appendix C.3.

---

### S-8 — Interceptor/envelope inspector API is completely absent

**Section:** §8.2, §17.2  
**Platforms:** All

§8.2 describes bus-level interceptors (Approach A) as the production logging/tracing path, and §17.2 describes a "live envelope inspector" for DevTools. But the interceptor API is never defined: no registration method, no signature, no removal, no ordering, no threading model, no observe-only vs mutate distinction.

**Fix:** Define the interceptor contract: `BusInterceptor { onPublish(topic: TopicRef, envelope: MessageEnvelope<*>): Unit }`. Define registration: `bus.addInterceptor(interceptor)` / `bus.removeInterceptor(interceptor)`. State that interceptors are **observe-only by default** (cannot mutate payload or block delivery). If mutation is ever allowed, specify the phase and forbid payload type changes.

---

### S-9 — Flutter + GoRouter: `RouteObserver` does not work for module scope

**Section:** §14.2  
**Platforms:** Dart/Flutter

The lifecycle table maps module scope to `RouteObserver / Navigator 2.0 route-aware`. But GoRouter (the spec's stated default router for Flutter) uses a declarative routing model where `RouteObserver` integration is limited and unreliable. A module-scope topology activated via `RouteObserver` will not correctly track GoRouter shell route enter/exit events.

**Fix:** Replace the Flutter module scope guidance with a GoRouter-specific mechanism: `ShellRoute` with a layout widget that activates the topology in `State.initState` and deactivates in `State.dispose`. Document this as the canonical module-scope host for GoRouter-based apps.

---

### S-10 — Flutter `APPLICATION_EAGER` startup: `WidgetsBindingObserver` does not trigger on launch

**Section:** §14.2  
**Platforms:** Dart/Flutter

`WidgetsBindingObserver` observes lifecycle events (paused, resumed, detached). It does **not** fire on startup. `APPLICATION_EAGER` topologies need to activate before the first frame, not when the app is paused/resumed.

**Fix:** Split the Flutter application-scope guidance into two: (a) startup activation in `main()` or app root `initState` before `runApp()` for `APPLICATION_EAGER`, and (b) `WidgetsBindingObserver` for ongoing lifecycle events (pause/resume handling). Document both.

---

### S-11 — Kotlin coroutine scope ownership for topology lifecycle is unspecified

**Section:** §14.2  
**Platforms:** Kotlin/Android

The reactive primitives table lists `Job.cancel()` for subscription disposal, but the parent `CoroutineScope` that owns the topology's `Job` is never specified. The choice matters: `viewModelScope` survives configuration changes, `lifecycleScope` does not; application-scoped topologies need a scope tied to the process, not a UI component.

**Fix:** Define the coroutine scope binding per lifecycle scope: `APPLICATION` → process-scoped `CoroutineScope(SupervisorJob() + Dispatchers.Default)`, `MODULE` → `NavBackStackEntry.lifecycle`-scoped or `ViewModel`-scoped, `PAGE` → `Lifecycle.repeatOnLifecycle(STARTED)` or `DisposableEffect`-equivalent.

---

### S-12 — Kotlin `NavBackStackEntry` module scope: `popUpTo` and multi-back-stack behavior unspecified

**Section:** §14.2  
**Platforms:** Kotlin/Android

Android's `NavBackStackEntry` lifecycle varies with navigation options. `popUpTo(inclusive=true)` destroys the entry; `restoreState=true` can recreate it. Multi-back-stack (bottom navigation) keeps multiple entries alive simultaneously. The spec says module scope is "activated when module is entered, deactivated when exited" but does not define behavior for these cases.

**Fix:** Specify the module scope owner as the **navigation graph's `NavBackStackEntry`** and document: (a) topology deactivates when entry is destroyed (including `popUpTo`), (b) topology reactivates when entry is recreated with `restoreState`, (c) multi-back-stack means multiple module-scope instances coexist (one per live entry).

---

### S-13 — SwiftUI application scope: `ScenePhase` re-evaluates on state changes

**Section:** §14.2  
**Platforms:** Swift/iOS

Using `ScenePhase` to detect application activation means the activation code runs every time `ScenePhase` changes value. `APPLICATION_EAGER` topology must activate exactly once, not on every scene transition.

**Fix:** Define the SwiftUI application-scope owner as a `@StateObject`-retained controller initialized in `@main App.init()` or via an `.environmentObject`. `ScenePhase` is for background/foreground lifecycle signals only, not for initial topology activation.

---

### S-14 — SwiftUI `NavigationPath` observation for module scope is too vague

**Section:** §14.2  
**Platforms:** Swift/iOS

"`NavigationPath` observation" is listed as the module-scope mechanism but provides no actionable contract. `NavigationPath` is a value type; detecting module enter/exit requires diffing path values before and after navigation events. The exact hook is not defined.

**Fix:** Define a concrete module-scope pattern for SwiftUI: a `NavigationStack` root view with an `@StateObject`-retained module controller activated in `.onAppear` and deactivated in `.onDisappear`, or a `NavigationSplitView` column coordinator using the same pattern. Specify that nested module paths require nested controllers.

---

### S-15 — Combine `ReplayTopic`: "Custom" implementation has no reference

**Section:** §14.1  
**Platforms:** Swift/iOS

The reactive primitives table lists "Custom or ReplaySubject (RxSwift)" for Combine's `ReplayTopic` backing. A custom Combine `ReplaySubject` is non-trivial to implement correctly (thread safety, subscriber management, buffer eviction). The spec provides no reference implementation.

**Fix:** Either (a) provide a reference `NidanaReplaySubject<T>` implementation in the `NidanaCore` Swift package, or (b) declare that pure-Combine mode (without RxSwift) requires the `NidanaCore` library to supply `ReplayTopic` backing and document this dependency explicitly. Option (a) is strongly preferred.

---

### S-16 — SSR hydration: no API, no protocol, no contract

**Section:** §15.6  
**Platforms:** TypeScript/Web

§15.6 mentions SSR as a web-specific concern: "StateTopic values can be embedded in the HTML payload, similar to how Redux serializes its store for SSR." But nothing is specified: which topics are serialized, server snapshot API, client `hydrate(snapshot)` timing, serialization format, escaping, hydration mismatch rules, or what happens when a topic's type has changed between server render and client hydration.

**Fix:** Add an SSR subsection to §15 defining: (a) `bus.snapshot(): Record<string, unknown>` on the server, (b) `<NidanaProvider snapshot={window.__NIDANA_STATE__}>` on the client, (c) dehydration happens after all eager topologies have activated and published, (d) hydration must complete before first `useTopic` render, (e) type mismatch falls back to `initial` with a dev-mode warning.

---

### S-17 — Angular `APPLICATION_EAGER` via `providedIn: 'root'` is lazy, not eager

**Section:** §15.3  
**Platforms:** TypeScript/Angular

The Angular lifecycle table maps `APPLICATION_EAGER` to "APP_INITIALIZER or root-level service constructor." But the example shows `@Injectable({ providedIn: 'root' })` with activation in the constructor. A `providedIn: 'root'` service is **lazily instantiated on first injection** — it is not eager. If nothing injects `AuthService` before the first component renders, the auth topology is not active yet.

**Fix:** Map `APPLICATION_EAGER` explicitly to `APP_INITIALIZER` (which runs before app bootstrap) or `ApplicationInitStatus`. Update the Angular example to show `APP_INITIALIZER` for eager topologies.

---

### S-18 — `BackWithResult(result: Any)` breaks the typed-topic contract

**Section:** §8.3  
**Platforms:** All

`NavIntent.BackWithResult(val result: Any)` carries an untyped payload. The receiving topology cannot safely downcast without prior knowledge of the type. This is the stringly-typed / weakly-typed problem the architecture is specifically designed to prevent, reintroduced through the navigation ADT.

**Fix:** Remove the payload from `BackWithResult`. The canonical result channel is a typed module-scoped `EventTopic<R>` published by the destination screen before navigating back. `BackWithResult` becomes `Back` with the result already on the topic. The `correlationId` links the intent to the result for traceability.

---

### S-19 — `DeepLink` executor drops query parameters and fragment

**Section:** §8.3  
**Platforms:** Flutter (example), all others by extension

`FlutterNavigationExecutor` implements `DeepLink(uri)` as `_router.go(uri.path)`, silently discarding query parameters and fragment. A deep link `myapp://checkout/cart?promo=SAVE10` navigates to `/checkout/cart` with no promo code.

**Fix:** Either (a) guarantee that `resolveDeepLinks` in the topology converts every `DeepLink` into a typed `GoTo(route, args)` with query parameters mapped to the `args` map — so the executor **never receives a `DeepLink`** — or (b) require the executor to pass the full URI when executing a deep link. Option (a) is architecturally cleaner and aligns with the "resolved" contract.

---

### S-20 — Envelope `source` for imperative `bus.publish()` calls is unspecified

**Section:** §2.4, §4.4  
**Platforms:** All

§2.4 defines `source` as `"topology:checkout-flow"` or `"service:payment-gateway"`. Services publish directly to topics via `bus.publish(topic, value)` — outside any topology. There is no specified way for the bus to know the source name for these publishes.

**Fix:** Add a publishing context API: either `bus.withSource("service:payment-gateway").publish(topic, value)` or a named publisher factory `val publisher = bus.publisher("service:payment-gateway"); publisher.publish(topic, value)`. Require all shell-boundary publishes to use this API in dev mode.

---

### S-21 — Deep link startup ordering: OS intent may arrive before navigation topology activates

**Section:** §8.1, §8.3  
**Platforms:** All

`APPLICATION_EAGER` topologies activate at startup. But OS deep link delivery (Android `Intent`, iOS `openURL`, web initial URL) may fire before or concurrently with topology activation. If the navigation topology is not yet active when the first deep link arrives, it is published to `NavigationTopics.intent` and immediately lost (EventTopic, no replay).

**Fix:** Add a startup contract: the navigation service must activate and subscribe to `NavigationTopics.intent` before OS intent delivery is enabled. For platforms where OS intents arrive before the bus initializes, the runtime adapter must buffer and replay the initial intent after activation. Alternatively, use a `ReplayTopic(1)` for `NavigationTopics.intent` to tolerate a one-event activation gap.

---

### S-22 — Topology definition stability: inline objects cause activation churn in declarative UIs

**Section:** §15.2, §15.4, §14.4 (SwiftUI/Compose)  
**Platforms:** React, Vue, SwiftUI, Jetpack Compose

In declarative UI frameworks, a topology object created inline in a component's body will have a new identity on every render. `useTopology(new CheckoutTopology())` in React will activate a new topology and deactivate the old one on every re-render.

**Fix:** Add a normative rule: "Topology definitions passed to lifecycle hooks must be **stable references** — declared as module-level constants, static singletons, or memoized. Implementations should warn in dev mode if a topology with a different object identity but the same `topologyId` is activated while one is already active."

---

### S-23 — "Never terminate a stream" is a rule without a mechanism

**Section:** §9.2, §2.3  
**Platforms:** All

§9.2 principle 2: "Never let an error terminate a stream." §2.3: "If a topology's pure transformer throws an unhandled exception, the underlying reactive framework will terminate that topology's subscription. The bus does not intercept developer exceptions." These two statements are in direct tension. The first is a rule; the second says the runtime will not enforce it.

**Fix:** Choose one model and state it clearly: (a) the bus provides no recovery — topology authors are solely responsible for preventing exceptions in transformer code, and the doc should say "never" means "you must prevent this yourself," or (b) the bus wraps every topology pipeline in a `catchError` handler that redirects exceptions to `ErrorTopics.appError` and continues the subscription. Document which is normative.

---

### S-24 — Circuit breaker state location contradicts the pure-wiring model

**Section:** §9.3  
**Platforms:** All

"Topology tracks failure count, stops retrying after threshold" implies mutable state inside the topology. But topology `declare()` bodies must not hold or mutate state — they are pure stream wiring. A failure counter inside a topology violates this invariant.

**Fix:** Show the circuit breaker as a `scan` accumulator over a failure-event stream: `failureEvents.scan(0, (count, _) => count + 1).distinctUntilChanged().filter(count => count < threshold)`. Make it explicit that all state is in the stream operator, not the topology body.

---

### S-25 — React `useSyncExternalStore` snapshot stability requires stable reference semantics

**Section:** §15.2  
**Platforms:** TypeScript/React

`useSyncExternalStore` requires `getSnapshot` to return the **same reference** when the value has not changed. If `bus.getCurrentValue(topic)` returns a new wrapper object on every call (e.g. by cloning or boxing the value), React will detect a "snapshot changed" on every render and loop into a re-render cycle.

**Fix:** Add a contract to `bus.getCurrentValue()` (or the React-specific `useSyncExternalStore` adapter): "Returns the exact same reference as the last `publish()` call. No cloning, wrapping, or boxing. The caller is responsible for value immutability."

---

### S-26 — `BackTo` is invalid for GoRouter; `popUntil` does not exist

**Section:** §8.3  
**Platforms:** Dart/Flutter

`FlutterNavigationExecutor` implements `BackTo(route)` as `_router.popUntil(route.path)`. GoRouter does not have a `popUntil` method.

**Fix:** Either (a) remove `BackTo` from the portable `NavIntent` ADT and mark it a platform-capability-dependent extension, or (b) provide a GoRouter emulation strategy: track navigation history manually and call `_router.pop()` in a loop until `currentRoute == target`, or use `_router.go(route.path)` with explicit documentation of the semantic difference (replace vs pop).

---

### S-27 — `BackWithResult` is absent from `FlutterNavigationExecutor` switch

**Section:** §8.3  
**Platforms:** Dart/Flutter

The `FlutterNavigationExecutor` example handles `GoTo`, `Replace`, `Back`, `BackTo`, `DeepLink`, `ShowModal`, and `DismissModal` — but has no branch for `BackWithResult`. Exhaustive switch on the ADT would fail to compile with it missing.

**Fix:** Add a `BackWithResult` branch. Per the fix in S-18, this should be `_router.pop()` (the result was already published to the result topic before this intent was emitted).

---

## 🟡 Minor

### M-1 — Conditional application scope is "open design question" in normative lifecycle text

**Section:** §6.1  
The spec describes "conditional application scope" as a valid pattern but also calls it "an open design question." Having an unresolved design question in the normative lifecycle section creates implementor uncertainty. **Fix:** Either specify it as a composition pattern (a meta-topology watching a condition topic), or move it to §17 Open Questions.

---

### M-2 — Module scope boundaries are app-defined, but this is not stated

**Section:** §6, §14.2  
The spec defines module scope as "activated when module is entered, deactivated when exited" without clarifying that the module boundary is **defined by the application and runtime adapter**, not auto-detected by the bus core. **Fix:** Add one sentence to §6.2: "Module scope boundaries are integrator-defined: they align with a navigation route group, feature entry point, or UI container as determined by the runtime adapter. The bus core has no concept of 'module'; it only activates and deactivates on explicit API calls."

---

### M-3 — Exception isolation granularity is ambiguous

**Section:** §2.3  
"An unhandled exception terminates the topology's subscription." A topology may have multiple pipelines. Does one thrown exception kill one pipeline or the entire topology activation? **Fix:** Define: an exception in one pipeline terminates **that pipeline's subscription** only; other pipelines in the same topology continue. Document whether the bus logs or republishes single-pipeline failures.

---

### M-4 — Scope violation detection references "topic scope annotations" that don't exist

**Section:** §11.4  
The automated verification pipeline promises CI detection of "page-scoped topology writing to app-scoped state topic." But topic scope annotations are never defined in the data model. **Fix:** Either add `scope: Scope?` to the `Topic` constructor (making topic scopes explicit), or remove this automated check from §11.4 and add it to §17 as a future capability.

---

### M-5 — `removeTopic()` cleanup semantics are undefined

**Section:** §6.6  
Manual topic removal (`bus.removeTopic(topic)`) is listed as a cleanup strategy with no postcondition: does the backing subject `complete()` (notifying active subscribers)? Does it `error()`? Are active topology subscriptions reading it torn down? **Fix:** Define: `removeTopic()` completes the backing subject (sends `onComplete` to all active subscribers), requiring topologies subscribed to that topic to handle stream completion gracefully.

---

### M-6 — OS permission guards inherit the wrong-combinator bug from the auth guard example

**Section:** §8.1  
§8.1 says OS permission guards use "the same pattern as auth guards." The auth guard pattern uses the wrong combinator (see C-8). Copying it for permission guards replicates the same spurious-re-navigation bug. **Fix:** Fix the auth guard example first (C-8), then confirm the permission guard section references the corrected pattern.

---

### M-7 — `resolvedIntent` should use a post-resolution ADT, not raw `NavIntent`

**Section:** §8.3  
After `resolveDeepLinks`, the resolved stream should never contain `DeepLink` variants — all deep links have been converted to typed `GoTo`/`Replace` intents. Using the same `NavIntent` ADT for both pre- and post-resolution streams allows `DeepLink` to reach the executor unanticipated. **Fix:** Introduce `ResolvedNavIntent` as a subset of `NavIntent` that excludes `DeepLink`. `resolvedIntent` is typed `EventTopic<ResolvedNavIntent>`, and the executor switch is exhaustive over the reduced set.

---

### M-8 — Navigation result cleanup lifecycle is undefined

**Section:** §8.3  
"Module-scoped result topic should be cleaned up when the flow completes" — but "flow complete" is never defined. If the source topology is torn down before it reads the result, the result is lost. If cleanup is too late, stale results leak into unrelated flows. **Fix:** Define: result topic is cleaned up when the source module scope exits (not when the destination exits). The `correlationId` on the result message is used to match it to the originating intent; only the correlated result is consumed.

---

### M-9 — `RouteTransition` type is referenced but not defined

**Section:** §8.3  
`NavigationTopics.history` carries `RouteTransition` values and `toRouteTransition` maps to them, but `RouteTransition` is never defined. **Fix:** Define `RouteTransition { from: RouteState, to: RouteState, intent: ResolvedNavIntent, timestamp: DateTime }` alongside the `NavigationTopics` definition.

---

### M-10 — §12.2 cross-platform portability strategy is aspirational without tooling contract

**Section:** §12.2  
Strategy 2 (Single-Language SSOT with code generation) says Kotlin or Dart pure transformers can be mechanically translated to other platforms. But no translation contract, type mapping, or acceptable-operator subset is defined. A Kotlin transformer using `data class` destructuring cannot be mechanically translated to Swift. **Fix:** Add a "portability subset" annex: a list of permitted types (primitives, sealed ADTs, lists, maps) and permitted operators (map, filter, combine, scan, withLatestFrom) that guarantee mechanical translatability. Anything outside this subset remains platform-specific.

---

## Recommended Fix Priority

Before the architecture is used to drive multi-platform implementations, the following must be resolved in order:

1. **C-4** — Settle subject creation timing model (declaration = IR; subjects created on activation)
2. **C-2** — Define `on()` semantics or remove it
3. **C-3** — Specify lazy topology registration and activation-then-deliver contract
4. **C-1** — Define envelope lineage rules for multi-input combinators
5. **C-8** — Fix navigation guard combinator (`withLatestFrom`)
6. **C-6, C-7** — Add `resolvedIntent` to `NavigationTopics`; define auth policy source
7. **C-9, C-10, C-11** — Define `currentRoute` producer, `toRouteTransition`, confirmed-navigation model
8. **S-7** — Define `TopologyDefinition` IR and `RecordingTopologyBuilder`
9. **S-5, S-6** — Define TestBus contract and scheduler injection API
10. **S-8** — Define interceptor registration and observer-only contract
11. **C-12** — Resolve Kotlin `StateFlow` duplicate-suppression conflict
12. **C-13, C-14** — Fix SwiftUI cancellable storage and React StrictMode contracts
13. **S-3** — Define bus acquisition and `NidanaProvider` equivalent for Flutter/Kotlin/Swift
14. **S-16** — Define SSR hydration protocol for TypeScript/Web
