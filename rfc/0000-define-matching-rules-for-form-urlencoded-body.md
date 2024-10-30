---
name: define-matching-rules-for-form-urlencoded-body
started: 2024-09-06
pr: pact-foundation/roadmap#0000
tracking_issue: pact-foundation/roadmap#0000
---
## Summary

This RFC proposes a way to define matching rules for body of a `application/x-www-form-urlencoded` request.

## Motivation

Currently we have the method [`match_form_urlencoded`](https://github.com/pact-foundation/pact-reference/blob/master/rust/pact_matching/src/form_urlencoded.rs#L12) to match the `application/x-www-form-urlencoded` request's body using matching rules.
But there is no way to define matching rules for the body to fully take advantage of this feature.

The only a way to define body for a `application/x-www-form-urlencoded` request is to define it as-is.

For example, we can define the request's body as `fullname=My+Full+Name&email=test%40example.com&password=abc%40123`.
Then the consumer **must** send the request with body exactly like this: `fullname=My+Full+Name&email=test%40example.com&password=abc%40123`, or
otherwise consumer's Pact test will failed, and the pact file will not be written.

That's because we can't define matching rules with this raw syntax. With the ability to define matching rules with the alternative syntax,
consumer will be able to send different values.

This example is focusing on request's body, but it also apply to response's body.

## Guide-level explanation

### With matching rules

When the RFC is implemented, the example usage would be:

```rust
let json = json!({
    "null": {
        "pact:matcher:type": "null"
    },
    "number": {
        "pact:matcher:type": "number",
        "value": 23.45
    },
    "string": {
        "pact:matcher:type": "type",
        "value": "example text"
    },
    "array": {
        "pact:matcher:type": "eachValue(matching(regex, 'value1|value2|value3|value4', 'value2'))",
        "value": ["value1", "value4"]
    }
});
pactffi_with_body(interaction, InteractionPart::Request, "application/x-www-form-urlencoded", json);
```

After the matching rules are extracted, the example body will be returned:

```query
number=23.45&string=example+text&array=value1&array=value4
```

This example body will be written into pact file, and will be used to send to provider to verify.

> Note: Null will be removed

### Without matching rules

We can also define body without matching rules:

```rust
let json = json!({
    "null": null,
    "number": 123,
    "string": "example value",
    "array": [null, -123.45, "inner text"],
});
pactffi_with_body(interaction, InteractionPart::Request, "application/x-www-form-urlencoded", json);
```

The example body will be extracted and look like this:

```
number=123&string=example+value&array=-123.45&array=inner+text
```

It will be exactly the same as we define the body using raw syntax:

```rust
let raw = "number=123&string=example+value&array=-123.45&array=inner+text";
pactffi_with_body(interaction, InteractionPart::Request, "application/x-www-form-urlencoded", raw);
```

### Null value

Null is simply ignored.
If null matching rule is defined, it will not do anything.
If consumer trying to request `null=something+else`, or provider trying to response `null=something+else`, we will got a mismatch:

```
Unexpected form post parameter 'null' received
```

This broke Postel's law. But it's not subject for this RFC.

An alternative approach is: We can instead move `null` into unsupported syntax list. Then `null` is not allowed to define.

### Unsupported syntax

Using json to define these values will result a panic:

- Boolean (true/false)
- Object
- Array of Arrays
- Array of Objects


For example, it will panic if we define like this:

* With matching rules:

```rust
let json = json!({
    "true": {
        "pact:matcher:type": "boolean",
        "value": true
    },
    "false": {
        "pact:matcher:type": "boolean",
        "value": false
    },
    "object": {
        "pact:matcher:type": "type",
        "value": {
            "key" => {
                "pact:matcher:type": "type",
                "value": "value"
            }
        }
    },
    "array_of_arrays": {
        "pact:matcher:type": "type",
        "value": [["value1", "value2"]]
    },
    "array_of_objects": {
        "pact:matcher:type": "type",
        "value": [{"key": "value"}]
    }
});
pactffi_with_body(interaction, InteractionPart::Request, "application/x-www-form-urlencoded", json);
```

* Without matching rules:

```rust
let json = json!({
    "true": true,
    "false": false,
    "object": {
        "key": "value"
    },
    "array_of_arrays": [["value1", "value2"]],
    "array_of_objects": [{"key": "value"}]
});
pactffi_with_body(interaction, InteractionPart::Request, "application/x-www-form-urlencoded", json);
```

## Reference-level explanation

Here is the flow we need to implement in Rust core (pact-reference project):

- Pact implementation call FFI method `pactffi_with_body` with:
    - Content type: `application/x-www-form-urlencoded`
    - Body: Integration JSON format , which may include:
        - Matching rules
        - Example values
        - Generators (is not supported in this RFC, resulted in consumer's test failed)
- Rust core will process the body:
    - If content type hint is `application/x-www-form-urlencoded` AND detected content type from body is `application/json`
        - Extract matching rules
        - Extract generators
        - Return example JSON body with example values e.g. `{ "key": "example value" }`
    - Example JSON body will be converted to example Form UrlEncoded body e.g. `key=example+value`
    - If JSON body has one of these cases, it will panic:
        - Boolean (true/false)
        - Object
        - Array of Arrays
        - Array of Objects
    - Register the interaction as normal with:
        - Extracted matching rules
        - Extracted generators
        - Example Form UrlEncoded body
        - Content type: `application/x-www-form-urlencoded`

### Unsupported syntax explaination

* Boolean: There is no standard way to represent boolean in query string. It can be 1/0, true/false or t/f. User need to define them explicitly.
* Object: There is no standard way to represent object in query string.
* Array of arrays: There is no standard way to represent array of arrays in query string.
* Array of objects: There is no standard way to represent array of objects in query string.

### Panic vs Warning

To prevent matching rules and generators of unsupported syntax to be written into pact file, Rust core need to panic.

If we log a warning and ignore the unsupported syntax, matching rules and generators of ignored query params will be written to the pact file. This is undefined behavior.

### Generators

Generators will be written to pact file. Consumer test will failed with this warning log:

```
Generators only support JSON and XML
```

User need to remove generators to make consumer test works.

## Drawbacks

We need to modify Rust core (pact-reference project)

## Rationale and alternatives

Alternative approaches:

### Hybrid syntax

```rust
let text = "number=matching(number, 123)&string=matching(regex, '\w\d', 'a1')&array=matching(eachValue(matching(regex, '\d{3}', '221')))"
pactffi_with_body(interaction, InteractionPart::Request, "application/x-www-form-urlencoded", text);
```

This is just a pseudocode. Parsing this syntax will be harder than Integration JSON format because we need to implement new code for it.

### Plugin

We can create a custom plugin to provide matching rules for `application/x-www-form-urlencoded` request.

I already create a POC for it at https://github.com/tienvx/pact-form-urlencoded-plugin

The problem with it are:

* We need to move matchers from Rust core to this plugin.
* The performance will be bad compare to implementing inside Rust core (pact-reference project)

## Unresolved questions

- There will be some bugs in Rust core need to be fix through the implementation of this RFC
- Generators will not be supported for now
- Postel's law need to be followed outside of the implementation of this RFC
- Due to lack of standards, some of the syntax will not be supported

## Future possibilities

### Define generators

Define generators for `application/x-www-form-urlencoded` response's body:

```rust
let json = json!({
    "id": {
        "pact:matcher:type": "regex",
        "regex": "^[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}$",
        "pact:generator:type": "Uuid"
    }
});
pactffi_with_body(interaction, InteractionPart::Response, "application/x-www-form-urlencoded", json);
```

Consumer will receive response's body like this from mock server:

```
id=46441a68-1d3d-40f2-bbec-6e9bd26a4047
```


Currently Rust core will throw this warning and do not generate values if we are trying to define generators in response's body:

```
2024-09-05T19:00:20.763719Z DEBUG tokio-runtime-worker pact_matching: Applying body generators...
2024-09-05T19:00:20.763730Z DEBUG tokio-runtime-worker pact_plugin_driver::catalogue_manager: Looking for a content generator for application/x-www-form-urlencoded
2024-09-05T19:00:20.764806Z  WARN tokio-runtime-worker pact_matching::generators::bodies: Unsupported content type application/x-www-form-urlencoded - Generators only support JSON and XML
```

### Support arrays and objects

Probably not a good idea, because there are no standard syntax, and each language handle it differently

Reference: https://blog.shalvah.me/posts/fun-stuff-representing-arrays-and-objects-in-query-strings

### Follow Postel's law

Following Postel's law, the provider may return fields that the consumer will just ignore. But if provider send extra param in body, we will got this mismatch:

```
Unexpected form post parameter 'extra' received
```

I believe this is a bug, and it happen independently with this RFC, we need to fix it in the future PRs.
