---
name: pact_mkii
started: 2026-07-30
revised: 2026-09-24
pr: pact-foundation/roadmap#146
---
## Summary

This RFC proposes a ground-up redesign of the Pact framework — working title **Pact MkII** — that keeps
Pact's core idea (consumer-driven contract tests that are captured as an artifact and replayed against the
provider) while replacing how the ecosystem is built and how matching is expressed. The proposal has five
pillars:

1. **One core, thin SDKs.** A single reference engine with a coarse-grained, versioned protocol replaces the
   current pair of lock-step implementations (pact-jvm and pact-reference/rust) and the fine-grained C FFI.
   Language SDKs become thin, largely generated clients of the engine.
2. **Declarative interactions, compiled plans.** Consumer DSLs produce a declarative *interaction
   specification*; the engine compiles it into an executable *matching plan* (the v2 matching engine already
   prototyped in pact-reference). Plans are inspectable — contract tests get an `EXPLAIN`.
3. **Everything is a component.** Plugins are not a bolt-on: transports, content handlers, matchers and
   generators — including the built-in HTTP/JSON ones — all implement the same component interfaces.
4. **Shapes with honest optionality.** The matching-rules model is replaced by a *shape language* that
   supports optional fields, discriminated unions, enums and cardinality — made safe by *variant testing*:
   every declared variation must actually be exercised by the consumer test and replayed by the verifier.
   Providers can publish their own response shapes, letting a *subsumption check* catch variance the
   consumer never declared before production does.
5. **Scriptable lifecycle.** Named hook points (auth injection, provider state setup, message
   production/consumption) are part of the core design, with declarative configuration first and scripts as
   the escape hatch.

A contract file remains the interchange artifact (a new, separately versioned format), the broker
workflow is unchanged, and existing v1–v4 pact files remain verifiable. Deterministic verification
remains the foundation; AI assistance is an optional layer, never a requirement.

**Revision 2 (2026-09-24): prototype evidence.** This revision incorporates the findings of
[Pact Janus](https://github.com/rholshausen/pact-janus), a prototype built to test the five pillars
against real code: one Rust engine, TypeScript and JVM SDKs, a CLI, three embeddings, a third-party
component and a conformance suite. All five pillars held. Where the evidence changed the design, the
text below says what we would now build, and the first revision is in this PR's history. The biggest
change is the engine embedding: the subprocess, not WASM, is every SDK's primary embedding. Every
unresolved question from the first revision now has an answer or a statement of what was learned; the
detail, with links to the ADRs and measurements behind each claim, is in
[RFC feedback](https://github.com/rholshausen/pact-janus/blob/main/Documentation/rfc-feedback.md).

## Motivation

Pact works, and is widely used, but twenty years of accumulated architecture is now the limiting factor —
for maintainers and for users.

**The maintenance model doesn't scale.** We maintain two complete, independent implementations (pact-jvm and
pact-reference/rust) that must be kept in lock-step, an FFI core wrapped by five-plus language libraries,
and wrappers around wrappers (e.g. Scala over pact-jvm). Each language has different maintainers and a
different release cadence, so features arrive years apart: plugin support shipped in 2022 for some
languages, while .NET only recently gained message interactions and its plugin support is still in
progress. Every new capability must be implemented, tested and released N times.

**The FFI leaks.** The shared core's C interface is very fine-grained, so each wrapper orchestrates it
differently and users moving between languages see different behaviour from "the same" core. Wrapper
authors must call cleanup functions to avoid leaks, terminate panics and exceptions at the boundary, and
bridge sync/async models by hand. These are exactly the classes of bugs a contract-testing tool should not
be generating.

**Plugins are a bolt-on.** The plugin architecture was retrofitted, so plugins integrate through a side
door: they are hard to write, harder to integrate into each language, and support is uneven across the
ecosystem.

**Optional values are Pact's most-cited gap.** The current answer — write a separate test per combination
(see the [FAQ](https://docs.pact.io/faq#why-is-there-no-support-for-specifying-optional-attributes)) — is
principled but unusable for realistic payloads with polymorphic values, flags and enums. The combinatorial
space overwhelms hand-written tests, so users either under-test or leave Pact.

**Matching behaviour is opaque.** When a match fails, users cannot see *how* the expected interaction was
evaluated. Matching rules are a map of path expressions to rules, with precedence and cascading semantics
that even maintainers must re-derive from the code.

The expected outcome of this RFC is a design the community can react to, followed by a tracking issue and a
staged implementation plan. The goal is that adding a capability to Pact becomes one change to one engine,
that every language gets it in the same release cycle with identical behaviour, and that the optionality
problem is solved without giving up Pact's example-based philosophy.

## Guide-level explanation

### What we keep, change and throw away

**Keep:**
- The consumer-driven workflow: expectations defined in consumer unit tests, captured into a pact file,
  replayed against the provider. This is Pact's identity and it is not negotiable.
- The pact file as a self-contained, brokerable artifact; the broker workflow, verification results and
  `can-i-deploy` semantics.
- The example-based philosophy: Pact tests behaviour with concrete examples, it is not a schema validator.
- Provider states (formalised, see below).
- Multi-protocol support: HTTP, synchronous RPC, and asynchronous messages with the transport abstracted.

**Change:**
- One engine, one behaviour: pact-jvm and the Rust core converge on a single reference engine; pact-jvm
  becomes a thin SDK like every other language.
- The engine boundary: a coarse-grained, versioned protocol (documents in, documents and events out)
  replaces the fine-grained C FFI.
- Matching: declarative interaction specs compiled to inspectable matching plans, replacing the
  matching-rules path map.
- Optionality: shapes plus variant testing replace "write a test per combination".
- The specification: from prose plus test cases to an *executable specification* — the protocol IDL, the
  plan semantics, and a compatibility suite every SDK must pass.

**Throw away:**
- The public fine-grained C FFI (`pact_ffi` as a public API).
- Duplicate full implementations of matching, mocking and verification in each ecosystem.
- Wrappers-of-wrappers as an architecture.
- Matching rules as the user-facing model (still readable for old pact files).
- Ad-hoc extension points (request filters as CLI flags, callback soup) in favour of named lifecycle hooks.

### A consumer test in Pact MkII

The DSL looks familiar, but note `optional`, `anyOf` and `oneOf` — and that the test closure receives a
*variant*:

```typescript
const getOrder = pact.interaction('get an order')
  .given('an order exists', { id: '42' })
  .request({ method: 'GET', path: '/orders/42' })
  .response({
    status: 200,
    body: json({
      id: integer(42),
      status: anyOf('PENDING', 'SHIPPED', 'DELIVERED'),
      shippedAt: optional(datetime('2026-07-30T10:00:00Z')),
      payment: oneOf('type', {
        card:    { type: 'card', last4: regex(/\d{4}/, '1234') },
        invoice: { type: 'invoice', dueDate: date('2026-08-30') },
      }),
      items: eachLike({ sku: string('SKU-1'), qty: integer(1) }, { min: 1 }),
    }),
  });

await pact.execute(getOrder, async (mock, variant) => {
  const client = new OrderClient(mock.url);
  const order = await client.getOrder('42');
  expect(order.lineCount).toBeGreaterThan(0);
});
```

The declared shape has a variant space: 3 statuses × `shippedAt` present/absent × 2 payment alternatives ×
2 list lengths (one item, and more than one: `eachLike`'s boundary variants) = 24 variants. The engine
samples this space (pairwise by default, exhaustive or explicit on request; the prototype covers it with
8) and
runs the test closure once per selected variant, with the mock server serving that variant. If the consumer
code blows up when `shippedAt` is absent, the test fails — *that* is what makes `optional` honest. The FAQ
objection to optional attributes has always been "optional means untested"; in MkII, declaring a variation
is a promise to exercise it, and the framework keeps the promise for you instead of asking you to write
twelve tests.

The pact file records the shape once, plus the concrete example for each exercised variant.

### Provider verification

The verifier is the same engine. For each interaction it replays every recorded request variant against the
provider and matches responses against the shape. Where today users bolt authentication on with request
filters, MkII has named hook points configured declaratively, with scripts as the escape hatch:

```yaml
# verifier.pact.yaml
provider: order-service
transports:
  http: { port: 8080 }
hooks:
  before-request:
    - use: oauth2-client-credentials     # built-in / component-provided
      with: { tokenUrl: "${AUTH_URL}", clientId: "${CLIENT_ID}" }
  state-setup:
    - exec: ./scripts/setup-state.sh     # or an HTTP endpoint, or a WASM hook
```

The same hooks exist on the consumer side and for messages (a `produce-message` hook replaces today's
per-language callback wiring). Verification runs identically from the CLI (`pact verify`) and from unit
tests, because both are the same engine invoked over the same protocol.

### Variance the consumer never declared

Variant testing makes *declared* variations honest, but replay-based verification has a blind spot in the
other direction: if the consumer declares a response with `status: 'PENDING'` and the provider happens to
return `PENDING` during verification, everything passes — yet in production, a combination of data and
downstream systems makes the provider return `SHIPPED`, which the consumer never tested. The verifier only
ever sees one sample of the provider's response per state; it cannot observe what the provider *could*
produce.

MkII addresses this with a second source of truth: the provider publishes its own **response shape** per
operation — authored, derived from its types (protobuf/OpenAPI import, serialiser reflection), or recorded
as the union of responses produced by the provider's own tests. The broker then runs a **subsumption
check**: every response the provider shape admits must be admitted by the shape the consumer actually
tested. The scenario above surfaces as a finding at `can-i-deploy` time:

```
✗ order-consumer is not compatible with order-service
  interaction 'get an order', response body $.status:
    provider may produce: 'PENDING' | 'SHIPPED' | 'DELIVERED'
    consumer has only tested: 'PENDING'
  interaction 'get an order', response body $.shippedAt:
    provider declares this field optional
    consumer has only tested it present
```

The fix on the consumer side is to widen the declaration — `anyOf('PENDING', 'SHIPPED', 'DELIVERED')`,
`optional(...)` — which variant testing then forces the consumer to actually exercise. The two mechanisms
form a loop: subsumption pushes consumers to declare the real response space, and variant testing proves
they handle it.

The scope here is deliberate: this catches variance the *consumer* team doesn't know about, using what the
provider team already knows or can derive from its types and tests. Variance that not even the provider
team knows about is an unknown-unknown; runtime observation ideas for that are noted under future
possibilities, but are not part of the core design.

### Seeing what matching will do

Because interactions compile to plans, you can inspect them (output from the existing v2 engine prototype
in pact-reference):

```
$ pact explain ./pacts/consumer-provider.json --interaction 'get an order'
:request (
  :method (
    #{'method == POST'},
    %match:equality ( 'POST', %upper-case ( $.method ), NULL )
  ),
  :path (
    #{'path == '/test''},
    %match:equality ( '/test', $.path, NULL )
  ),
  :body (
    %if (
      %match:equality ( 'text/plain', $.content-type, NULL,
        %error ( 'Body type error - ', %apply () ) ),
      %match:equality ( 'Some nice bit of text', %convert:UTF8 ( $.body ), NULL )
    )
  )
)
```

After a verification failure, `pact explain --executed` shows the same plan annotated with the actual
values and which nodes failed — matching stops being a black box.

### What this means for each audience

- **Users** get identical behaviour in every language, optional/polymorphic payloads that are actually
  testable, inspectable matching, and first-class auth/state hooks.
- **Language SDK maintainers** stop reimplementing Pact. An SDK is: generated protocol bindings + an
  idiomatic DSL and test-framework integration (JUnit annotations, pytest fixtures, Jest helpers) + the
  conformance suite. New engine features surface in an SDK by regenerating bindings and adding DSL sugar.
- **Plugin authors** implement the same component interfaces the built-in functionality uses, so the
  extension path is the well-trodden path, documented by the core's own source.
- **Existing users** keep their pact files: the MkII engine reads and verifies v1–v4 pacts, and
  `pact upgrade` converts v3/v4 files to the new format, reporting anything the conversion narrows or
  loses. Teams migrate consumer-by-consumer; the broker mediates mixed fleets as it does today.

## Reference-level explanation

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│ Language SDKs (thin): DSL + test-framework integration     │
│   pact-js · pact-jvm · pact-go · pact-python · pact-net …  │
└──────────────┬─────────────────────────────────────────────┘
               │ Engine Protocol (versioned IDL, coarse-grained)
┌──────────────▼─────────────────────────────────────────────┐
│ Pact Engine (single reference implementation, Rust)        │
│  kernel: session lifecycle · plan compiler · plan          │
│          interpreter · pact model (v1–v4 read, v5 r/w)     │
│  components (same interfaces, in-tree or third-party):     │
│    transports: http, grpc, message …                       │
│    content: json, xml, form, protobuf, text/binary …       │
│    matchers/generators: core set, plugin-provided          │
│    hooks: oauth2, exec, wasm-script …                      │
└────────────────────────────────────────────────────────────┘
```

#### The engine boundary

The Engine Protocol is defined once, as schema-governed JSON documents ("frames") carried over a frozen,
one-function byte pipe. JSON Schema is the IDL; WIT describes only the pipe. The schemas are the
specified surface, a CI checker enforces their open-world evolution rules, and SDK bindings are
generated from them
([ADR 0002](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0002-document-first-protocol-over-frozen-pipes.md)). The protocol is
delivered over three embeddings, all speaking it identically:

1. **Subprocess** (primary, for every SDK): a `pact-engine` executable speaking the protocol over stdio
   with Content-Length framing (the LSP model). It is spawned per test run by the SDK, pinned by protocol
   version in its handshake, and exits when its stdin closes, so it cannot outlive its host. It is never a
   shared daemon. Spawn-to-ready is about a millisecond, and a call costs almost nothing over in-process.
2. **WASM component** (offline operations): the same engine compiled to a WASM component, for hosts that
   want Pact's answers without a native binary — `explain`, `upgrade`, the subsumption check, variant
   enumeration — in a broker, an IDE, a browser, or a sandboxed CI step. It runs the engine's own work
   within 10–35% of native.
3. **Minimal C ABI** (fallback, not prototyped): three functions — create, call-with-document, free —
   carrying the same frames, for embedders that need neither of the above.

The first revision made WASM the preferred embedding. The prototype showed that its WASM engine cannot
run a consumer test or a verification in any language: the mock's exchange loop and HTTP server run on
threads, and a `wasm32-wasip2` guest has none. (It has sockets; the thread is the blocker.) It also
cannot host third-party components, since a WASM guest cannot host WASM. Performance was never the problem; capability was
([ADR 0023](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0023-the-subprocess-is-the-primary-embedding-and-wasm-serves-offline-operations.md),
[performance report](https://github.com/rholshausen/pact-janus/blob/main/Documentation/performance-report.md)). The cost is that per-OS native binaries come back,
distributed the way esbuild and Biome ship theirs through npm. There are two routes back to a WASM
engine that runs a test, both named and not taken: WASI 0.3 (`wasm32-wasip3`), whose component-model
async would let the exchange loop run as a thread-free async task, and which the Rust project is
promoting to a supported target; or a transport the host provides. The decision is to be reassessed when
the first is available.

The protocol is *coarse-grained and document-oriented*: an SDK submits a complete interaction specification
in one call, rather than orchestrating dozens of stateful mutations. Sketch:

```
consumer-session:
  create(config) -> session
  add-interaction(session, interaction-spec) -> handle | structured-error
  start-transport(session, transport, options) -> endpoint
  variants(session, handle) -> list<variant>          // for variant-driven test loops
  serve-variant(session, handle, variant-id)
  finalise(session) -> { per-interaction results, pact-file }

verification:
  verify(source, target, options) -> stream<event>    // hooks and progress are events
  explain(source, interaction, options) -> plan-text

lifecycle:
  everything returns errors as values; panics cannot cross the boundary;
  async is event/stream based; sessions are the only resource and are
  closed by finalise (no per-object cleanup calls)
```

This directly removes the FFI failure modes: there is no per-object memory management, no
panic-across-boundary, no bespoke async bridging, and — because the orchestration lives inside the engine —
no room for two SDKs to sequence primitives differently and get different behaviour.

Two things the sketch still leaves in the SDK, and should not. First, only the SDK knows whether the
consumer's test closure *passed* for a variant. Today every SDK must withhold a contract whose test failed,
and one that forgets writes a dishonest contract. The sketch needs an operation reporting that verdict to
the engine. Second, engine-side failures only reach the SDK at `finalise`, so they surface at the end of a
suite rather than in the test that caused them. Both are open.

#### Interaction specifications and the plan compiler

An interaction spec is a declarative document (JSON) containing: description, provider states, the
request/response or message parts, and for each part a **shape** — the merged expected structure with
matching semantics attached inline, replacing the parallel matching-rules map. The engine compiles a spec
to a matching plan; the plan interpreter executes plans against actual values via value resolvers (HTTP
request, HTTP response, message). This is the v2 matching engine already prototyped in
`pact-reference/rust/pact_matching/src/engine` (plan nodes: containers, actions, values, resolvers,
pipelines; interpreter; pretty/executed forms).

Design consequences:

- The **plan node grammar and core action set** (`match:equality`, `match:regex`, `expect:empty`,
  `convert:UTF8`, …) become specified, versioned surface. Grammar versions are ordered, the engine says
  which it reads, and anything a component contributes declares which it targets, so every mismatch
  fails by name before anything runs. Components contribute custom actions under namespaces, invoked by
  the interpreter, rather than implementing matching end-to-end.
- *How much* of a plan a component contributes is open. The prototype let a content component contribute
  a whole slot's plan fragment. That works, but it makes the component a shape compiler for everything in
  the slot, and without knowing the variant: a pinned variant checked against a fragment passed where the
  engine's own plan correctly failed. The likely answer is contribution per operator: the engine compiles
  the slot, and the component says what one value operator means for its content type
  ([ADR 0022](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0022-a-fragment-replaces-its-slots-plan-and-declares-a-grammar-the-engine-says-it-reads.md),
  [stress test](https://github.com/rholshausen/pact-janus/blob/main/Documentation/plan-fragment-stress-test.md)).
- Old pact files are supported by compiling matching rules to plans; the plan compiler is the single place
  where v1–v4 cascading/precedence semantics live.
- `explain` is a kernel operation, not a feature each SDK builds.

#### The shape language

Shapes generalise today's matchers with structural operators:

| Operator | Meaning | Variant dimension? |
|---|---|---|
| `type/regex/datetime/…` | today's matchers, unchanged | no |
| `optional(shape)` | may be absent; must match when present | yes: present / absent |
| `nullable(shape)` | value or null (distinct from absent) | yes |
| `anyOf(v1, v2, …)` | enum of allowed values | yes: one per value |
| `oneOf(discriminator, {alt: shape…})` | discriminated union / polymorphism | yes: one per alternative |
| `eachLike(shape, {min, max})` | today's array matching + cardinality | boundary variants (min, min+1, and max when finite) |
| `forbidden` | must not be present (e.g. PII assertions) | no |

**Variant semantics** are the heart of the optionality answer:

- The engine computes the variant space of each interaction and selects a covering sample — **pairwise by
  default**, exhaustive below a threshold, and always including any variant the user pins explicitly.
- **Consumer side**: the test closure runs once per selected variant with the mock serving that variant.
  A variant that the consumer code cannot handle fails the build. Only exercised variants are recorded.
- **Provider side, requests**: the verifier replays every recorded request variant; the provider must
  accept all of them. This is why requests need no provider shape. It depends on request dimensions
  being **pinned** on the consumer side: under a variant the mock accepts only requests that variant
  admits, so a declared request width is one the consumer actually sent. The cost is that a closure over
  such an interaction is variant-*parameterised*: it reads the variant to decide what to send.
- **Cardinality points are matched as regions.** `min` and `max` are exact; `min+1` is served as `min+1`
  elements and matched as "more than the minimum", so a basket of five meets it
  ([ADR 0024](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0024-request-dimensions-stay-pinned-and-a-cardinality-point-matches-a-region.md)).
- **Provider side, responses**: the provider's actual response is matched against the shape; optional
  fields match whether present or absent, and `oneOf` matches via the discriminator. Where a specific
  response variant must be induced (e.g. "the shipped order" case), the variant can pin provider state
  parameters: `given('an order exists', { shipped: whenVariant('shippedAt', 'present') })`. The DSL
  helper expands to a separate `variant-params` member — never a wrapped value, which would be ambiguous
  against real parameter data — mapping each point of a dimension to a value. The engine resolves it
  per variant, and a provider that cannot produce a state reports `state-unavailable`, distinct from
  `failed`, because the remedy is a contract change
  ([ADR 0009](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0009-variant-bound-provider-state-parameters.md)).

This preserves Pact's discipline — nothing is declared that is not demonstrated — while collapsing the
dozen hand-written tests into one test with a managed matrix. Crucially, a shape is still not a schema:
it is only valid alongside the examples that exercised it.

#### Provider shapes and the subsumption check

Consumer-declared shapes only cover what consumers thought of; replayed verification samples one provider
response per state and therefore cannot detect latent provider variance. To close this, a provider may
publish a **provider shape** for each operation, and the broker (or the engine locally) checks
*subsumption*: `admits(provider shape) ⊆ admits(consumer tested shape)` for each interaction, where
`admits` is the set of values a shape accepts. Shapes were designed to make this decidable — it is a
structural walk comparing value spaces, not schema-format gymnastics.

Findings are asymmetric by design (Postel's law is preserved):

- **Extra fields** the provider may produce are fine — consumer shapes are must-ignore by default — unless
  the consumer marked them `forbidden`.
- **Wider value spaces** are findings: provider enum/union larger than the consumer tested; provider type
  broader (e.g. number where consumer tested integer); provider `nullable` where consumer tested only
  non-null.
- **Weaker presence** is a finding: provider declares a field optional that the consumer only tested
  present (the classic "nullable column" production break).
- The reverse direction (consumer requests) is already covered by variant replay, so no provider-side
  request shape is needed.

A provider-shape entry is matched to a consumer interaction by its description and provider states,
or, failing that, **by operation**. An entry may carry a *selector*: shapes over request slots, such as
a method and a path pattern, matched against the requests the consumer recorded. A provider whose shape
comes from its types has no way to know the words a consumer team used to describe an interaction.
Matching on them alone silently checked nothing, and the report read like reassurance
([ADR 0025](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0025-a-provider-shape-entry-may-select-interactions-by-operation.md)).

Provider shape provenance, in decreasing order of fidelity:

1. **Recorded from the provider's own tests**: the engine records the union of response shapes the
   provider's unit tests produce (a "provider self-contract") — highest honesty, since it is
   demonstrated, not asserted.
2. **Derived from types**: protobuf descriptors, OpenAPI documents, or serialiser reflection. This is
   the bi-directional-contracts idea, made shape-native. A shape derived from a hand-written OpenAPI
   document found exactly what a recorded one found. One derived from an ORM-generated document found
   4.5× as much, all of it true and almost none of it useful: one finding per nullable column
   ([spike 7.3](https://github.com/rholshausen/pact-janus/blob/main/spikes/7.3-type-derived-shapes/FINDINGS.md)).
3. **Authored** by the provider team.
4. **Observed**: accumulated from responses seen across verification runs.

The check runs wherever compatibility is decided: `can-i-deploy` combines verification results with the
subsumption result. Subsumption failures **warn by default**, for decided findings and manual reviews
alike, and a team can raise either to block. Exemptions are scoped by field, interaction or consumer,
require a reason, and may expire
([ADR 0016](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0016-subsumption-defaults-to-warn-with-mandatory-reason-exemptions.md)).
Providers that publish no shape simply get today's semantics — replay-only verification — so the
mechanism is adoptable per-provider.

Where the check cannot decide, it says `unknown` and asks for review; it never guesses. Every operator
has a comparability class. Literals, kinds, presence, unions and cardinality are decided by set
containment. Regexes and date formats are decided on identity, against a plain string, and when the
provider admits a finite set of values the consumer's own matcher can test; two different regexes are
`unknown`. An operator a component contributes is `unknown` unless the two shapes are identical, or the
component declares how to compare it.

#### Components (plugins as the core design)

The kernel knows nothing about HTTP or JSON. It loads components implementing four interfaces:

- **transport**: start/stop a mock endpoint; drive requests at a provider; map wire messages to/from the
  abstract interaction parts. (http, grpc, message-broker adapters, websockets…)
- **content**: for a content type — parse, canonicalise, contribute plan fragments, generate values.
  (json, xml, form-urlencoded, protobuf, text, binary…)
- **matcher/generator**: named plan actions beyond the core set.
- **hook**: implementations for named lifecycle points (`before-request`, `state-setup`,
  `produce-message`, `consume-message`, `after-verification`…).

Built-in components are compiled into the engine but implement the same interfaces, so "writing a plugin"
means implementing the interfaces the core uses. The prototype tested this: a third party wrote a `text/csv`
component from the published specifications without reading engine source, and the same `.wasm` ran in a
consumer test and in verification
([third-party component report](https://github.com/rholshausen/pact-janus/blob/main/Documentation/third-party-component-report.md)). Third-party components are WASM
components by default (portable, sandboxed, no per-OS binaries). Hosting them is a capability of the native
embeddings, since a WASM-embedded engine cannot host WASM. Transports that need raw sockets or
long-lived servers can run out-of-process, speaking the engine protocol's own stdio framing: today's
pact-plugins *architecture*, without a second wire format. Out of process, the environment a component
sees is enforced, and its filesystem and network grants are documented intent only. Components are
distributed as OCI artifacts of their own type, declared in project config and pinned by digest; the
engine resolves, caches and re-verifies them, so a pinned second run fetches nothing.

Two privileges remain for the built-ins, and the design should name them. They are compiled in rather
than loaded. And the kernel, although it knows no HTTP vocabulary, still assumes HTTP's *structure* in
two places: parts named `request` and `response`, and a mismatch answered as a `500`. Both need to go
before a message transport can be built.

#### The executable specification

The Pact specification becomes three enforceable artifacts, replacing prose-plus-test-cases:

1. the Engine Protocol IDL (versioned),
2. the plan grammar and core action semantics (versioned, with golden plan/result corpora),
3. the compatibility suite (grown from `pact-compatibility-suite`) that every SDK must pass in CI.

An SDK is *conformant* when it passes the suite against a pinned engine version. Because SDKs are thin,
conformance mostly tests DSL-to-spec translation rather than matching behaviour. In the prototype, both
SDKs pass all 47 cases of one shared corpus in their own language, and the JVM SDK, written from the
specification alone, records the same contract as the TypeScript one, member for member.

#### Generated, AI-assisted SDKs

Each SDK has three layers: (a) protocol bindings — fully generated from the IDL; (b) the idiomatic layer —
DSL, test-framework integration, docs; (c) the conformance suite. The idiomatic layer is maintained from a
canonical *SDK specification* (behavioural spec plus per-language style guide) with AI agents doing the
mechanical regeneration when the spec changes, and language maintainers reviewing. The guarantee of
consistency is the conformance suite, not the generation method — AI assistance lowers the maintenance
cost, it is not load-bearing for correctness. This is how one team can plausibly keep eight SDKs current
within one release cycle. The prototype's two SDKs have hand-written layers of 723 lines (TypeScript)
and 1,802 (Java), and a CI audit fails the build if matching logic creeps back in. A specification
change was carried into both by agents that never saw each other's code, and both passed the suite
([thinness audit](https://github.com/rholshausen/pact-janus/blob/main/Documentation/thinness-audit-report.md)).

#### The contract file

The first revision called this "Pact file format v5". The prototype names its artifact separately (a
Janus contract), so that the Pact specification stays free to define its own next version, and the name
is for the community to decide
([ADR 0011](https://github.com/rholshausen/pact-janus/blob/main/Documentation/decisions/0011-contracts-as-self-identifying-json-documents.md)). "v5" below means
that artifact, whatever it is called.

- JSON, self-contained, broker-compatible: today's broker already stores, dedupes, diffs and answers
  `can-i-deploy` for it. Provider shapes have no broker resource yet
  ([broker integration notes](https://github.com/rholshausen/pact-janus/blob/main/Documentation/broker-integration-notes.md)).
- Per interaction: description, provider states (typed parameters), transport binding, parts with
  **shape + exercised example variants**, component requirements (e.g. `content/protobuf >= 2`).
- The engine reads v1–v4 and writes v5; `pact upgrade` converts v3/v4 to v5 (matching rules become shapes;
  the single example becomes the sole variant), and reports every place the conversion loses or narrows
  something. Downgrade is intentionally unsupported. A converted pact is a blunter consumer document than
  a native one: a field with no matching rule becomes exactly the value its example held, so a pact
  that saw `SHIPPED` records `SHIPPED`, never the values the consumer tolerates. The subsumption loop is sharper for consumers who declare their shapes.
- Mixed fleets work through the broker: new consumers publish v5; providers need an MkII verifier to verify
  v5 pacts, but the MkII verifier also verifies all old pacts, so providers upgrade first at no cost.

#### Migration path

1. Engine ships with v1–v4 read/verify support from day one; providers can switch verifiers immediately.
2. SDKs ship MkII as a new major version; the old DSL surface is kept where it maps cleanly (a
   compatibility facade), so most consumer tests need mechanical changes only.
3. `pact upgrade` and broker-side content negotiation cover the artifact layer.
4. The existing FFI and pact-jvm cores enter maintenance mode once their SDKs pass the conformance suite
   on the new engine.

### AI-assisted verification (optional layer)

Everything above is deterministic and requires no AI. On top of it, two optional capabilities are worth
designing for:

- **Mismatch diagnosis**: `pact explain --executed` output is a structured trace that is ideal input for an
  LLM to summarise ("the provider renamed `shippedAt` to `shipped_at` in the invoice variant").
- **Agentic verification**: a mode where the verifier emits, from the pact file, a task description that an
  AI agent uses to stand up/configure the provider, satisfy provider states, and run verification —
  useful where state setup automation doesn't exist yet. The *judgment* of pass/fail always remains with
  the deterministic engine; the agent only performs orchestration.

Both are additive; no part of the core workflow depends on them.

## Drawbacks

Each drawback the first revision predicted is annotated with what the prototype found
([detail](https://github.com/rholshausen/pact-janus/blob/main/Documentation/rfc-feedback.md#4-the-drawbacks-measured)).

- **This is a very large undertaking** for a volunteer-driven ecosystem: engine, protocol, format, five-plus
  SDKs, tooling, docs. Staged delivery is mandatory and even then it is multi-year.
- **Osborne effect**: announcing MkII may stall adoption and contribution to current Pact before MkII is
  ready. A Python-2/3-style community split is a real risk if migration is not near-mechanical. *The
  mitigation is real: the prototype's engine agrees with the pact specification's own test cases on
  583 of 583 in scope, and verifies existing pacts unchanged.*
- **Native binaries return.** *This replaces the first revision's "WASM host maturity varies", and it is
  worse than predicted.* The limit was not the host languages but the WASM guest, which cannot serve a
  mock or drive a provider. So the subprocess is the primary embedding, and per-OS `pact-engine`
  binaries must be distributed for every SDK. That is the pain that made pact-ruby-standalone unpopular.
  What keeps it from repeating that experience is the lifecycle: per test run, exit on stdin EOF, pinned
  by protocol version. The prototype tested that lifecycle on Linux and on real Windows.
- **Social cost**: pact-jvm ceasing to be an independent implementation displaces maintainer identity and
  the redundancy benefits of two implementations (bugs caught by divergence).
- **New specified surface**: the plan grammar becomes public, versioned API; getting its stability
  guarantees wrong would be costly. *The versioning policy held under a stress test. What strained was
  how much of a plan a component may contribute (see the plan compiler above).*
- **Variant testing has sharp edges**: consumers' test closures must be variant-agnostic or
  variant-parameterised; careless shapes can still explode the sampled matrix; pairwise coverage is a
  heuristic, not a proof. *Confirmed. Whether a failing variant can be identified depends on the test
  framework: JUnit 5 and Vitest name it for free, and a flat loop names nothing. Request-side variants
  make the closure variant-parameterised. And the engine cannot tell whether the closure handled a
  response, only the SDK can. Pairwise stayed small on this RFC's example: 8 variants cover all 24.*
- **Subsumption findings can overwhelm**: provider shapes derived from types tend to overstate the real
  response space (every field nullable in the ORM ≠ every field absent in practice), so a strict policy
  would drown teams in findings and teach them to rubber-stamp. Warn-first defaults and good provenance
  guidance are essential. *Confirmed and measured at 4.5×, and warn-first is now the default. Converted
  v1–v4 pacts add noise from the other direction, because they are too narrow.*
- **A contract can say less than a v1–v4 pact said by default.** *New.* v1–v4 accept
  `application/json; charset=utf-8` where the consumer wrote `application/json`. A shape can say that
  only through an operator the HTTP component contributes, which does not exist yet, so an upgraded
  contract is stricter. Stricter is the safe direction, and still wrong for the commonest header there is.
- **Non-JSON content strains a JSON-shaped document model.** *New.* The prototype's CSV component
  showed that the document model has no member order, that encoding an empty list loses the columns, and
  that a component's decode errors reach the user as ordinary mismatches.

## Rationale and alternatives

- **Coarser FFI instead of a protocol** (keep `pact_ffi`, make it handle+JSON based): improves memory and
  behaviour-divergence issues but keeps native binary distribution, panic boundaries and async bridging.
  Retained anyway as embedding #3 — but as a transport for the one protocol, not a separate API.
- **Shared daemon** (the ruby-standalone model): history shows the pain is process lifecycle, port
  management and version skew. MkII's subprocess mode is per-test-run, spawned by the SDK, and
  version-pinned by the protocol. The first revision called it the fallback. The prototype made it the
  primary embedding, and this lifecycle is why that is acceptable.
- **WASM as the primary embedding** (the first revision's choice): no native binaries, sandboxed,
  in-process. Rejected on evidence, not preference: a `wasm32-wasip2` guest has no threads, and the
  engine's mock runs on them, so it cannot run a mock or a verification as built. It is kept for offline
  operations, and reassessed when WASI 0.3 lands.
- **Schema-based contracts** (OpenAPI/bi-directional as the core model): solves optionality by giving up
  Pact's central guarantee — that the consumer demonstrably works against what it declares. Shapes plus
  variant testing get schema-like expressiveness while keeping the guarantee.
- **Incremental evolution of the status quo**: every motivation item worsens with ecosystem growth; the
  .NET feature lag and the FFI divergence bugs are structural, not accidental. The cost of lock-step
  reimplementation is already the dominant tax on the project.
- **Full per-language rewrites with a common spec** (Pact today, but tidier): the compatibility suite helps,
  but it demonstrably has not kept implementations in step; the economics of N implementations don't change.

## Unresolved questions

The first revision listed twelve. The prototype answered eight, partly answered one, learned
something about the other three, and raised three new ones. Full answers with evidence are in
[RFC feedback](https://github.com/rholshausen/pact-janus/blob/main/Documentation/rfc-feedback.md#2-the-unresolved-questions).

**Resolved by the prototype:**
- *Components day one, or HTTP/JSON privileged?* Day one for the interfaces. The built-ins are privileged
  only in packaging, plus two HTTP-shaped assumptions in the kernel that are named above.
- *IDL and WASM-host story?* JSON Schema documents over a frozen WIT pipe. The subprocess is the
  primary embedding in every language, and WASM serves offline operations.
- *Variant sampling defaults?* Exhaustive below a threshold, then a named deterministic pairwise
  algorithm. Boundary variants and pins are always included, and an exceeded budget fails rather than
  truncating.
- *Provider-state/variant linkage?* A separate `variant-params` member resolved per variant. A state
  the provider cannot reach is `state-unavailable`.
- *Subsumption warn or block, and exemption scoping?* Warn by default. Exemptions are scoped by field,
  interaction or consumer, need a reason, and may expire.
- *Subsumption decidability?* Comparability classes per operator; `unknown` is a review, never a guess.
- *Plan grammar stability and versioning?* Ordered grammar versions, declared by both sides, so every
  mismatch fails by name. How much of a plan a component contributes is still open (below).
- *Performance of WASM vs native FFI?* The engine beats `pact_ffi` on every like-for-like scenario except
  large request bodies, and WASM is within 10–35% of native. Capability, not performance, decided the
  embedding.

**Partly resolved:**
- *Broker handling of the new artifacts.* Contracts store in today's broker. Provider shapes need a new
  resource, and verification results need per-consumer/provider-pair counts.

**Still open:**
- Naming and versioning, and governance and funding: for the community. The prototype frames them in its
  staged implementation plan.
- Message interaction hook design details (sync message RPC, broker adapters): designed, not built.
- How a component contributes to a plan: per slot, as prototyped, or per operator, as the evidence
  suggests.
- The protocol operations that let the engine, not each SDK, know whether a consumer's test passed.
- Whether an array admits the empty list by default.

**Out of scope here, addressable later:**
- Deprecation timeline for current implementations.
- Broker API evolution beyond artifact acceptance.

## Future possibilities

- **Semantic contract diffing**: with shapes, `pact diff` can answer "did the provider's contract surface
  change in a breaking way?" structurally, enabling better `can-i-deploy` explanations.
- **Property-based provider fuzzing**: shapes are generators; the verifier could probe providers beyond
  recorded variants.
- **Bi-directional convergence**: provider shapes derived from OpenAPI/protobuf (provenance #2 above) are
  the first step; a fully provider-driven workflow — consumers verified against a published provider shape
  with no provider-side replay at all — is a natural extension.
- **Consumer-side runtime guard**: the consumer's tested shape can be embedded in the consumer application
  to log or emit a metric when a production response falls outside tested territory — an early-warning
  signal that reuses the same artifact.
- **Runtime drift observation**: for unknown-unknowns — variance not even the provider team knows about —
  a provider-side observer could summarise production responses into shapes (values abstracted, no
  payloads retained) and diff them in the broker against declared and tested shapes. Deliberately excluded
  from the core design: it needs a production component and privacy guarantees, and the subsumption check
  covers the knowable cases.
- **Centrally executed verification**: a broker/platform that runs the engine itself, since verification is
  fully described by pact file + verifier config + hooks.
- **Stateful interaction sequences** (sagas, websockets, streaming): plans and transports were designed with
  multi-step exchanges in mind.
- **A WASM engine that runs tests**: a transport the *host* provides, with an exchange loop the host drives,
  is the one route to a WASM engine that can serve a mock. It would make the in-process, binary-free
  embedding the first revision wanted possible again, at the cost of transport code in every SDK.
- **Deeper AI integration**: agentic provider onboarding and mismatch triage as described above, once the
  deterministic core is stable.
