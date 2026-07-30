---
name: pact_mkii
started: 2026-07-30
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

The pact file remains the interchange artifact (a new major format version), the broker workflow is
unchanged, and existing v1–v4 pact files remain verifiable. Deterministic verification remains the
foundation; AI assistance is an optional layer, never a requirement.

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

The declared shape has a variant space: 3 statuses × `shippedAt` present/absent × 2 payment alternatives =
12 variants. The engine samples this space (pairwise by default, exhaustive or explicit on request) and
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
  `pact upgrade` converts v3/v4 files to the new format. Teams migrate consumer-by-consumer; the broker
  mediates mixed fleets as it does today.

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

The Engine Protocol is defined once in an IDL (WIT and/or protobuf — to be settled during implementation)
and delivered over three embeddings, all speaking the same protocol:

1. **WASM component** (preferred): the engine compiled to a WASM component, hosted in-process
   (wazero for Go, Chicory for JVM, wasmtime bindings for .NET/Python, built-in support in Node). No
   process management, no native binaries per platform, sandboxed, memory-safe.
2. **Subprocess**: a `pact-engine` executable speaking the protocol over stdio/socket (the LSP model), for
   platforms without a good WASM host and for the CLI itself.
3. **Minimal C ABI** (fallback): three functions — create, call-with-document, free — carrying the same
   protocol messages, for embedders that need neither of the above.

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
  `convert:UTF8`, …) become specified, versioned surface. Content handlers and plugins contribute plan
  *fragments* and custom actions under namespaces, rather than implementing matching end-to-end. A plugin
  that can express its matching as a plan fragment needs no runtime callback for matching at all;
  plugin-supplied actions are invoked by the interpreter where a fragment is insufficient.
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
| `eachLike(shape, {min, max})` | today's array matching + cardinality | boundary variants (min, min+1) |
| `forbidden` | must not be present (e.g. PII assertions) | no |

**Variant semantics** are the heart of the optionality answer:

- The engine computes the variant space of each interaction and selects a covering sample — **pairwise by
  default**, exhaustive below a threshold, and always including any variant the user pins explicitly.
- **Consumer side**: the test closure runs once per selected variant with the mock serving that variant.
  A variant that the consumer code cannot handle fails the build. Only exercised variants are recorded.
- **Provider side, requests**: the verifier replays every recorded request variant; the provider must
  accept all of them.
- **Provider side, responses**: the provider's actual response is matched against the shape; optional
  fields match whether present or absent, and `oneOf` matches via the discriminator. Where a specific
  response variant must be induced (e.g. "the shipped order" case), the variant can pin provider state
  parameters: `given('an order exists', { shipped: whenVariant('shippedAt', 'present') })`.

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

Provider shape provenance, in decreasing order of fidelity:

1. **Recorded from the provider's own tests**: the engine records the union of response shapes the
   provider's unit tests produce (a "provider self-contract") — highest honesty, since it is
   demonstrated, not asserted.
2. **Derived from types**: protobuf descriptors, OpenAPI documents, or serialiser reflection, imported by a
   content component. This is the bi-directional-contracts idea, made shape-native.
3. **Authored** by the provider team.
4. **Observed**: accumulated from responses seen across verification runs.

The check runs wherever compatibility is decided: `can-i-deploy` combines verification results with the
subsumption result. Whether a subsumption failure blocks deployment or warns is a broker policy decision
(teams adopting incrementally will want warn-first). Providers that publish no shape simply get today's
semantics — replay-only verification — so the mechanism is adoptable per-provider.

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
is documented by reading the core. Third-party components are WASM components by default (portable,
sandboxed, no per-OS binaries); transports that need raw sockets or long-lived servers can run
out-of-process over gRPC, which is essentially today's pact-plugins model retained as the escape hatch.
Components are distributed as OCI artifacts and declared with versions in project config; the engine
resolves, caches and verifies them.

#### The executable specification

The Pact specification becomes three enforceable artifacts, replacing prose-plus-test-cases:

1. the Engine Protocol IDL (versioned),
2. the plan grammar and core action semantics (versioned, with golden plan/result corpora),
3. the compatibility suite (grown from `pact-compatibility-suite`) that every SDK must pass in CI.

An SDK is *conformant* when it passes the suite against a pinned engine version. Because SDKs are thin,
conformance mostly tests DSL-to-spec translation rather than matching behaviour.

#### Generated, AI-assisted SDKs

Each SDK has three layers: (a) protocol bindings — fully generated from the IDL; (b) the idiomatic layer —
DSL, test-framework integration, docs; (c) the conformance suite. The idiomatic layer is maintained from a
canonical *SDK specification* (behavioural spec plus per-language style guide) with AI agents doing the
mechanical regeneration when the spec changes, and language maintainers reviewing. The guarantee of
consistency is the conformance suite, not the generation method — AI assistance lowers the maintenance
cost, it is not load-bearing for correctness. This is how one team can plausibly keep eight SDKs current
within one release cycle.

#### Pact file format v5

- JSON, self-contained, broker-compatible.
- Per interaction: description, provider states (typed parameters), transport binding, parts with
  **shape + exercised example variants**, component requirements (e.g. `content/protobuf >= 2`).
- The engine reads v1–v4 and writes v5; `pact upgrade` converts v3/v4 to v5 (matching rules become shapes;
  the single example becomes the sole variant). Downgrade is intentionally unsupported.
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

- **This is a very large undertaking** for a volunteer-driven ecosystem: engine, protocol, format, five-plus
  SDKs, tooling, docs. Staged delivery is mandatory and even then it is multi-year.
- **Osborne effect**: announcing MkII may stall adoption and contribution to current Pact before MkII is
  ready. A Python-2/3-style community split is a real risk if migration is not near-mechanical.
- **WASM host maturity varies** by language; the subprocess embedding mitigates but reintroduces process
  management (the pain that made pact-ruby-standalone unpopular), even if per-test-run and protocol-versioned.
- **Social cost**: pact-jvm ceasing to be an independent implementation displaces maintainer identity and
  the redundancy benefits of two implementations (bugs caught by divergence).
- **New specified surface**: the plan grammar becomes public, versioned API; getting its stability
  guarantees wrong would be costly.
- **Variant testing has sharp edges**: consumers' test closures must be variant-agnostic or
  variant-parameterised; careless shapes can still explode the sampled matrix; pairwise coverage is a
  heuristic, not a proof.
- **Subsumption findings can overwhelm**: provider shapes derived from types tend to overstate the real
  response space (every field nullable in the ORM ≠ every field absent in practice), so a strict policy
  would drown teams in findings and teach them to rubber-stamp. Warn-first defaults and good provenance
  guidance are essential.

## Rationale and alternatives

- **Coarser FFI instead of a protocol** (keep `pact_ffi`, make it handle+JSON based): improves memory and
  behaviour-divergence issues but keeps native binary distribution, panic boundaries and async bridging.
  Retained anyway as embedding #3 — but as a transport for the one protocol, not a separate API.
- **Shared daemon** (the ruby-standalone model): history shows the pain is process lifecycle, port
  management and version skew. MkII's subprocess mode is per-test-run, spawned by the SDK, and
  version-pinned by the protocol — and it is the fallback, not the primary embedding.
- **Schema-based contracts** (OpenAPI/bi-directional as the core model): solves optionality by giving up
  Pact's central guarantee — that the consumer demonstrably works against what it declares. Shapes plus
  variant testing get schema-like expressiveness while keeping the guarantee.
- **Incremental evolution of the status quo**: every motivation item worsens with ecosystem growth; the
  .NET feature lag and the FFI divergence bugs are structural, not accidental. The cost of lock-step
  reimplementation is already the dominant tax on the project.
- **Full per-language rewrites with a common spec** (Pact today, but tidier): the compatibility suite helps,
  but it demonstrably has not kept implementations in step; the economics of N implementations don't change.

## Unresolved questions

**Through the RFC process:**
- Naming and versioning: is this Pact specification v5 + "Pact 6" SDK majors, or a new brand (MkII)?
- Is "everything is a component" a day-one architecture or a target (i.e. may HTTP/JSON be kernel-privileged
  initially)?
- IDL choice for the Engine Protocol (WIT vs protobuf vs both) and the WASM-host story per language.
- Variant sampling defaults: is pairwise the right default? What are the caps and overrides?
- Provider-state/variant linkage design (`whenVariant` above is a sketch).
- Subsumption policy: should a failed check warn or block `can-i-deploy` by default, and how do teams
  scope exemptions (per field, per interaction, per consumer)?
- Subsumption decidability limits: comparing two regex matchers or two datetime formats for inclusion is
  possible but expensive/fiddly — where does the check degrade to "unknown, review manually"?
- Governance: who owns the engine, the SDK spec, and conformance sign-off? What is the funding model?

**Through implementation:**
- Plan grammar stability guarantees and its versioning policy.
- Broker/PactFlow handling of v5 artifacts (rendering shapes, diffing, matrix semantics for variants).
- Performance envelope of WASM embeddings vs today's native FFI.
- Message interaction hook design details (sync message RPC, broker adapters).

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
- **Deeper AI integration**: agentic provider onboarding and mismatch triage as described above, once the
  deterministic core is stable.
