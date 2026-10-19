---
name: generator_ordering
started: 2026-10-07
pr: pact-foundation/roadmap#0000
tracking_issue: pact-foundation/roadmap#0000
---

## Summary

Apply the generators of a body in a defined order: **by path depth, shallowest first**.
This change makes generators that has shallow depth (usually change the *shape* of a value) work in a predictable way.

## Motivation

The [`RandomArray` plugin][plugin] expands a one-item array template into a random number of
items between a `min` and a `max`:

```json
"generators": {
  "body": {
    "$.items":          { "type": "RandomArray", "min": 2, "max": 4 },
    "$.items[*].name":  { "type": "RandomString", "size": 10 },
    "$.items[*].price": { "type": "RandomInt", "min": 1, "max": 100 }
  }
}
```

Today the order is arbitrary (map iteration), so the generated values are unexpected and
unpredictable.

When the generators are applied in the order `$.items[*].name` → `$.items` → `$.items[*].price`,
the generated body looks like this:

```json
{ "items": [
  { "name": "qomZtbtirs", "price": 42 },
  { "name": "qomZtbtirs", "price": 17 },
  { "name": "qomZtbtirs", "price": 88 }
] }
```

When they are applied in the order `$.items[*].name` → `$.items[*].price` → `$.items`:

```json
{ "items": [
  { "name": "qomZtbtirs", "price": 42 },
  { "name": "qomZtbtirs", "price": 42 },
  { "name": "qomZtbtirs", "price": 42 }
] }
```

If the generator for `$.items` is applied first, the generated values are as expected:

```json
{ "items": [
  { "name": "qomZtbtirs", "price": 42 },
  { "name": "kHnQpLwGxv", "price": 17 },
  { "name": "TbfsaEimYd", "price": 88 }
] }
```

## Guide-level explanation

Nothing changes for pact authors: the same pact file, the same generators, and now the
expected body on every run — in the mock server, and in the request the verifier sends to the
provider.

Existing pacts keep working, because core generators write to disjoint paths and never depended
on each other's order:

```json
"generators": { "body": {
  "$.id":   { "type": "Uuid" },
  "$.name": { "type": "RandomString", "size": 10 }
} }
```

generates the same kind of body before and after this change.

There is nothing to teach: users do not choose an order, and no pact file format or plugin API
changes.

## Reference-level explanation

Before applying the generators of a scope (JSON, XML and form-urlencoded alike), sort them by
path depth:

```rust
entries.sort_by_key(|(key, _)| key.len());
```

Nested expansions run outermost first:

```json
"generators": { "body": {
  "$.orders":                    { "type": "RandomArray", "min": 2, "max": 2 },
  "$.orders[*].items":           { "type": "RandomArray", "min": 3, "max": 3 },
  "$.orders[*].items[*].name":   { "type": "RandomString", "size": 5 }
} }
```

```json
{ "orders": [
  { "items": [ { "name": "aBcDe" }, { "name": "fGhIj" }, { "name": "kLmNo" } ] },
  { "items": [ { "name": "pQrSt" }, { "name": "uVwXy" }, { "name": "zAbCd" } ] }
] }
```

Corner cases, by example:

- **Overlapping paths** — parent first, child wins:

  ```json
  "generators": { "body": {
    "$.foo":     { "type": "ProviderState", "value": "${data}" },
    "$.foo.bar": { "type": "Uuid" }
  } }
  ```

  parent first → `{"foo": {"bar": "<uuid>"}}`; child first → `{"foo": {"bar": "<from state>"}}`.

- **Same depth** — order unspecified, and irrelevant:

  ```json
  "generators": { "body": {
    "$.a": { "type": "Uuid" },
    "$.b": { "type": "Uuid" }
  } }
  ```

## Drawbacks


## Rationale and alternatives

- Add new concept: **A `generator-category` catalogue field**:

  ```json
  { "generator": "RandomArray", "generator-category": "structure" }
  ```

  `generator-category` only accept pre-defined value (`structure`, `data`). Default value is `data`.
  Apply `structure` generators first, then `data` generators.
- **Hard-coding generators like `RandomArray` in the core**: does not scale to future plugins.
- **Doing nothing**: the shape-changing generators such as `RandomArray` generator stays unusable.

## Unresolved questions

- Is the `ArrayContains` affected by this change?
- I'm not sure the **Overlapping paths** corner case above is the correct way of using generators.

## Future possibilities

- Other shape-changing plugins under the same rule (object expansion, tree generation).

[plugin]: https://github.com/tienvx/pact-random-array-plugin
