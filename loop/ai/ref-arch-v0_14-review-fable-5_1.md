# Nidana Bus Reference Architecture v0.14.0: Review

**Reviewer:** Claude Fable 5.1
**Date:** 2026-10-09
**Reviewed:** `arch/nidana-bus-ref-arch-v0_14_0.md` (3,928 lines) plus `loop/human/ref-arch-v0_14_0-commentaries.txt`
**Companion:** `loop/ai/ref-arch-v0_14-map-fable-5_1.html` (bird's-eye map, per-section scores, links into the source)

Finding IDs (F1..F7) and editorial IDs (E1..E8) below are the same ones used in the map.

---

## Verdict

The core is sound and has converged. Topic as typed key, topology as inert IR interpreted by the bus, envelope lineage rules, publish-free observers with capability gating, ref-counted module scope, module-local topology identity: these sections (§2, §3, §4, §6.5, §7.3, §9.2, §9.3) are implementable as written and I would not spend another prose iteration on them.

Worth continuing: yes, but not as v0.15 of this document. The diff from v0.13.2 to v0.14.0 is 290 lines, mostly a rename (Interceptor to Observer) and one new subsection (§4.6). You are past the point where the document improves by rereading it. Fourteen versions and zero running code is the actual risk, not the stream bias you suspect in yourself. The frontend world already converged on reactive primitives (Flow, Combine, RxJS, Signals); betting on streams is the least controversial part of this design.

On novelty, be precise when you pitch it. No single idea here is new. The combination is unusual for frontend: an inert topology IR that yields a static graph, CI-enforced invariants (single-writer, acyclic, scope) with a runtime first-claim as defense in depth, and lifecycle scopes that are explicit rather than inferred. The nearest relatives, which §17 does not name, are Kafka Streams (`Topology.describe()` is the same static-graph idea, and you borrowed the word), Cycle.js (drivers as shell, `main` as pure), redux-observable and NgRx Effects (epics as stream topologies), and Square Workflow. Cite them; it makes the "what is actually new" claim smaller and more defensible.

Two things hold the document back. First, five internal contradictions (F1 to F5) that an implementer would hit in the first week; one of them (F1) runs through the flagship example. Second, the document specifies the output of a platform company (four platforms, seven layers each, a tooling roadmap with "shipped v1" labels) while being a one-person project with a stale Dart spec targeting v0.10.2.

---

## Settled: do not touch

- §2.1 to §2.4: topic variants, envelope fields, lineage rules for multi-input combinators.
- §3.1, §3.2, §3.4 to §3.7, §3.9, §3.10, §3.12: identity, subject lifecycle, reentrancy normalization, envelope threading, scheduler contract, observer model, idempotency, thread safety, dedup.
- §4.1 to §4.4, §4.6: typed references, uniqueness levels, registry organization, module-local topology identity.
- §6.4, §6.5: sequencer anti-pattern, topology as data. §6.5 is the strongest idea in the document.
- §7.3, §7.4, §7.6: module scope binding with grace period, topic vs topology lifecycle, Initializing variant.
- §9.2, §9.3, §10.1, §10.2, §11.2, §13.1, §14, §18, §20, §22.

---

## Findings that block implementation

### F1. Single-writer is defined for topologies only, and the document's own shell code violates it

§3.11 (L744) enforces "exactly one writing topology per StateTopic" via AST scan over topology bodies and runtime first-claim at activation. Imperative publishes from shells are not counted. Then the document's own examples publish to `auth.state` from the shell: `PersistenceService` (L499), `FirebaseAuthService` (L2912), `LegacyAuthBridge` (L3189), and Appendix B draws Auth Service, Persistence Service, and `auth-core` all touching `auth.state`. The "races eliminated by construction" row in §11.1 (L2182) rests on single-writer, so for the recommended hydration pattern the claim does not hold. §11.2 (L2202) admits this in one clause.

Options:
1. Shell publishers claim writes. `bus.publisher("service:persistence", writes = [AuthTopics.state])` participates in first-claim and in the CI scan. Cost: one more declaration per service; the manifest for binary modules already exists for this.
2. Shells never write StateTopics. Hydration publishes `auth.hydrated` on an EventTopic; the `auth-core` reducer owns `auth.state`. Cost: one more topic per hydrated state, but it matches §6.4's own reducer advice.
3. Downgrade the §11.1 row to "requires discipline".

Recommendation: 1 and 2 together. Option 1 closes the hole structurally; option 2 is the pattern you show in §2.5 instead of the current one.

**Decision (2026-10-09): 1 and 2, no exceptions.** One owner per `StateTopic`, topology or claimed shell publisher; everyone else publishes events named by intent; a one-line last-write-wins owner covers trivial state; runtime first-claim is the guarantee and the CI scan is the early warning. Applied in `arch/nidana-bus-ref-arch-v0_14_1.md` (§1.1, §2.1, §2.4, §2.5, §3.11, §5.2, §5.4, §6.4, §9.4 executor, §11.1, §11.2, §12.6, §18.3, §20.2, Appendix A, B, D).

### F2. Topics have no scope, yet three places treat them as scoped

§7.4 and §7.5 say topics live until `removeTopic`; only topologies have scopes. But the §7.1 diagram (L1357) nests topics inside Module and Page scopes, §7.2 (L1430) speaks of an "APPLICATION-scoped state topic", and §9.4 result handling (L2040) says "the result topic is module-scoped and cleaned up when its scope exits". The D.1 scope-violation lint inherits the confusion.

Decide one of: (a) topics gain an optional scope and are removed when it ends, which conflicts with the no-auto-GC rule in §3.2; (b) the scope lint is restated as "a topology must not write a StateTopic whose owning topology has a longer scope", with topics unscoped, and §9.4's stale-result sentence is deleted (an EventTopic retains nothing anyway). Recommendation: (b).

### F3. `APPLICATION_LAZY` has no activation mechanism

§7.1 (L1399) says lazy topologies activate "on first read or write to a topic the topology handles". Nothing says how the bus knows a lazy topology's topic set before activation (it can: `buildDefinition()` is pure, but registration is never specified), what happens when the first interaction is a publish to an EventTopic (the event fires into a cold subscription and is lost), or what `bus.registerLazy(...)` looks like. This was C-3 in the v0.10 review and is still open. The "conditional activation via a meta-topology with switchMap" paragraph (L1410) also contradicts §6.1: a topology activating other topologies is an effect inside a topology body.

Recommendation: either specify `register(topology)` for lazy scope with an "activate then deliver" rule for EventTopic triggers, or drop `APPLICATION_LAZY` from v1 and let services defer their own expensive work behind a StateTopic. The second is cheaper and loses little.

### F4. Correlation threading through a page's imperative publish is asserted, not specified

§9.4 result handling (L2040) claims the destination page's publish "carries the same correlationId as the originating GoTo intent (the bus threads this)". The page publish is a shell root publish via `bus.publisher`; §3.5 threads correlation inside pipelines, not through a platform router and back into a page. Either add `publisher.publish(topic, value, causedBy = envelope)` to §2.4 and show how the page obtains the envelope, or delete the sentence.

### F5. Formal claims overreach

§12.1 (L2240): "the order of topology activation does not change the data flow result". False for EventTopic (§7.4 L1466: events before subscription are lost) and for lazy scope. True for StateTopic-only graphs; say so. §12.2 determinism excludes time operators but not `switchMap` into effectful service streams (§6.3 L1211), which is the main way services enter topologies. Scope the claim to "pure transformers, non-time operators, and inputs on topics".

---

## Findings to decide

### F6. Platform facts under a normative heading

§3.3 (L551) says RxDart subjects deliver synchronously. RxDart subjects are `sync: false` by default; synchronous delivery needs `sync: true` at construction. §3.3 and §15.5 treat `MutableStateFlow` as an ordering problem; it is also a conflation problem (intermediate values dropped), which dedup masks for StateTopic but not for a `MutableSharedFlow`-backed ReplayTopic with slow collectors. Both are fixable in a sentence, but §3 is marked normative, so an implementer will trust it. Verify each row of §15.5 against the library you actually pick.

### F7. Scope of the document

Four platforms, six engine variants, three web frameworks, seven layers each, and a D.2 table that says "shipped v1" for tooling that does not exist. Decide what v1 is. My recommendation: v1 is Dart, everything else is informative. Freeze §15, §16, Appendix C, and Appendix D as "informative, unvalidated" and move them out of the spec (see commentary 6).

### F8. TypeScript dedup default silently diverges from the other platforms

§3.12 (L766) makes `StateTopic` dedup default-on, with TypeScript using reference equality unless an `equals` parameter is supplied. Transformers return fresh immutable objects on every emission, so on TypeScript dedup never fires by default while on Kotlin, Dart and Swift it does. The same topology then rerenders on one platform and not on the others, which is exactly the portability break §3.4 goes to lengths to prevent elsewhere. Decide: require `equals` on every TypeScript `StateTopic` (lint), or make structural equality the default and accept the cost. I lean to requiring `equals`.

---

## Decisions applied in v0.15.0 (2026-10-09)

All findings below were decided and baked into v0.15.0, which is split into three documents under `arch/`: `nidana-bus-ref-arch-spec-v0_15_0.md` (normative), `nidana-bus-ref-arch-rationale-v0_15_0.md`, and `nidana-bus-ref-arch-guide-v0_15_0.md`. Sections are renumbered per document; cross-document links carry the document name.

- **F2, option (b).** Topics stay unscoped. The lint is now scope compatibility: a `StateTopic`'s owner must have a scope at least as long as every topology that reads it. Publishers that claim a topic declare their scope. Retention across owner re-activation is defined through `reduceInto`, which seeds the fold with the topic's current value. §9.4's stale-result sentence is gone. (Spec §7.1, §7.2, §6.3; Guide §3.6, Appendix C.)
- **F3.** `APPLICATION_LAZY` is dropped; scopes are `APPLICATION`, `MODULE`, `PAGE`. Deferred initialization is documented as a shell pattern: the service subscribes eagerly, claims a readiness `StateTopic`, loads the expensive resource on the first request, and drains pending requests when ready. The topology side stays `APPLICATION`-scoped, so the static graph is complete and no first event is lost. (Spec §7.1 "Service Topology Scopes", with code.)
- **F4.** The correlation claim is deleted. Results are matched on a request id carried in the intent and echoed in the result; matching is a `filter` on payload. The observability variant (`RouteState` carrying the causing intent's correlation) is parked in Guide §10.2. (Guide §3.6.)
- **F5.** New Spec §3.13 Startup Barrier: `bus.start()` ends bootstrapping; publishes before it are queued, so activation order is irrelevant for every topic variant within `APPLICATION` scope. §12.1's claim is scoped to `StateTopic` plus the barrier, with the intermediate-topic caveat; §12.2 names stream factories as inputs and states per-engine determinism within a tick. The deep-link bootstrap now uses the barrier. (Spec §3.13, §7.2, §11.2; Rationale §3.1, §3.2; Guide §3.5.)
- **F6.** Fixed in the sentence: RxDart subjects are constructed with `sync: true`; `StateFlow` conflation is called out and `EventTopic`/`ReplayTopic` use `MutableSharedFlow` with `extraBufferCapacity`. (Spec §3.3; Guide §6.1, §6.5.)
- **F7.** v1 is Dart/Flutter only. The abstract says so; the Guide's preface marks other platforms as unvalidated design targets; "shipped" became "planned" throughout the tooling roadmap. (Spec Abstract; Guide About, Appendix C.)
- **F8.** `equals` is required on every TypeScript `StateTopic`; the factory's type rejects a declaration without it, and codegen emits a structural `equals` per contract type. (Spec §3.12; Guide Appendix C.3.)
- **E1 to E8 and commentaries 1 to 3.** Stale draft references removed; appendix mentions linked; `topology()` declared as sugar for the class form; "load-bearing" gone; the dead §19.6 pointer fixed; the dangling envelope-context sentence removed; the circuit breaker now resets on success; §17 gained Kafka Streams, Cycle.js, redux-observable / NgRx Effects and Square Workflow; shared vs module-local code blocks are labelled; §6.1's enforcement status is stated and listed in the verification map.

## Editorial

| ID | Item | Where |
|---|---|---|
| E1 | References to earlier drafts and versions (your commentary 4) | L792, L1518, L1891, L2007, L2230, L3462 |
| E2 | Unlinked "Appendix C/D" mentions (your commentary 5) | L722, L748, L1281, L1334, L1549, L1774, L2052, L2403, L2735, L3041, L3369 |
| E3 | Two DSL styles: `topology("...") { }` (14 uses) vs class with `declare()` (8 uses). Pick one; the class form is what §15.3 shows per platform | throughout |
| E4 | "load-bearing" | L1658 |
| E5 | §9.3 points to §19.6 "on retention"; §19.6 is internationalization and says nothing about retention | L1776 |
| E6 | §6.2 promises a "wrapping topology pattern" for envelope-aware transformers and never shows it (your commentary 3) | L1188 |
| E7 | §10.3 circuit breaker: prose says successes reset the counter; the code filters failures only and never resets | L2151 to L2168 |
| E8 | §17 omits the closest relatives (Kafka Streams, Cycle.js, redux-observable / NgRx Effects, Square Workflow) | L2873 |

---

## Your commentaries, answered

**1. Global vs module-local code mixed in one snippet.** Agree. Separate blocks are one fix; a cheaper one that scales across 40 snippets is a one-line header comment convention stated once in §4.4 and used everywhere: `// shared contracts (Layer 1)` vs `// checkout module (local)`. Sites: §2.5 Persistence Pattern (`AuthTopics` shared, `PersistenceService` local), §4.2, §9.4 (`NavigationTopics` shared, `nav-resolver` and executor in the navigation package).

**2. Are the §6.1 body restrictions enforceable?** Not structurally, today. Two partial mechanisms exist: the RecordingBuilder runs the body at `buildDefinition()` time, so I/O in a body would execute during the CI scan (detectable as a smell, not prevented), and D.1 "pure-substrate purity" is a v2 import-graph lint that covers transformers, not bodies. The one type-level move available: make `read()` return an opaque handle with no `.value` accessor, which removes the most tempting violation (synchronous current-value read). Everything else is review discipline; it belongs in the developer guide, and D.5 should say so.

**3. Example of a transformer that needs envelope context?** No, there is none, and I would not add one. Delete the sentence at L1188. Tracing inside a transform is what the observer seam is for; exposing the envelope to a transformer breaks the portability argument in §3.5 and §13.1. If you want an escape hatch, define `readEnveloped(topic): Stream<MessageEnvelope<T>>` explicitly as giving up purity, in §15, not in §6.2.

**4. Remove references to previous versions.** Six sites, listed in E1. One nuance: the rationale behind "considered and rejected" items (auto-GC in §3.2, TTL cleanup in §7.5) is worth keeping; only the historical framing goes. Write "Auto-cleanup is rejected because..." rather than "was rejected in earlier drafts".

**5. Hyperlink Appendix references.** Eleven sites, listed in E2. Mechanical.

**6. Split into two documents?** Yes, but three, not two, and the spec is the only one an implementer (human or AI) must read.

| Document | Contents | Size |
|---|---|---|
| Normative spec | §2, §3, §4, §6, §7, §9.2, §9.3, §10.1, §10.2, §3.11 enforcement rules, glossary | roughly 1,200 lines |
| Rationale and positioning | §1, §11, §12, §13, §14, §17, §18, §22 | roughly 900 lines |
| Implementer's guide | §8, §9.4 as a reference topology, §15, §16, §19, §20, §21, Appendix C, D | the rest |

Trade-off: three files drift unless each spec section carries a stable ID that the other two cite. The alternative, one file with each section tagged normative or informative, is cheaper to maintain but leaves the implementer reading 4,000 lines to find 1,200. Given your stated goal (an AI implements from this), take the split. A side benefit: today §3 says "normative" and §7, which is equally contractual, does not. The split forces that decision for every section.

---

## Recommendation: the next version is code

1. Fix F1 to F5 in v0.14.1. They are a few paragraphs each, and F1 changes one example pattern.
2. Do the split (commentary 6) and the editorial pass (E1 to E8). Stop there with prose.
3. Retarget the Dart spec from v0.10.2 to the new normative spec. Build `nidana_dart_core`, `nidana_dart_topology`, and `TestBus` as pure Dart with no Flutter dependency. The §21.1 TestBus contract is the prerequisite for everything else; promote it out of "open questions".
4. Use Appendix B as the acceptance scenario. The first thing that scenario will tell you is whether F1's resolution survives contact with hydration, SDK adapters, and bridges all at once.
5. Only then decide whether Kotlin, Swift, or TypeScript happen at all.

The ideas are good enough to be falsified by code. Another document revision cannot do that.
