---
name: define-matching-rules-for-form-urlencoded-body
started: 2024-09-06
pr: pact-foundation/roadmap#99
tracking_issue: pact-foundation/roadmap#109
---
## Summary

This RFC proposes defining matching rules for bodies of type `application/x-www-form-urlencoded`.

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

This example body (along with matchers) will be written into pact file.

### Without matching rules

We can also define body without matching rules:

```rust
let json = json!({
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

### Generators

Generators can be defined like this:

```rust
let json = json!({
    "id": {
        "pact:matcher:type": "regex",
        "regex": "^[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}$",
        "pact:generator:type": "Uuid"
    },
    "age": {
        "pact:matcher:type": "integer",
        "value": 0,
        "max": 130,
        "min": 0,
        "pact:generator:type": "RandomInt"
    },
    "name": [
        {
            "pact:matcher:type": "regex",
            "regex": "Mr\\.|Mrs\\.|Miss|Ms\\.",
            "value": "",
            "pact:generator:type": "Regex"
        },
        {
            "pact:matcher:type": "type",
            "value": "",
            "size": 10,
            "pact:generator:type": "RandomString"
        }
    ]
});
pactffi_with_body(interaction, InteractionPart::Response, "application/x-www-form-urlencoded", json);
```

Consumer will receive response's body like this from mock server:

```
id=46441a68-1d3d-40f2-bbec-6e9bd26a4047&age=65&name=Mr.&name=Ea8XIXh7NQ
```

### Unsupported syntax

These values are not supported:

- Null: There is no way to represent null in query string. User may want to define empty string instead.
- Boolean (true/false): There is no standard way to represent boolean in query string. It can be 1/0, true/false or t/f. User need to define them explicitly.
- Object: There is no standard way to represent object in query string.
- Array of Arrays: There is no standard way to represent array of arrays in query string.
- Array of Objects: There is no standard way to represent array of objects in query string.


#### With matching rules:

```rust
let json = json!({
    "null": {
        "pact:matcher:type": "null"
    },
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

These matchers and values are ignored. Unsupported error messages are logged. The extracted example body will be empty.

#### Without matching rules:

```rust
let json = json!({
    "null": null,
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

The values are ignored. Unsupported error messages are logged. The extracted example body will be empty.

### Special cases

- keys with no values (`a=&b=&c=`): can be achieved by define empty string value
- values with no keys (`=a&=b&=c`)
    - can be defined by json
    - can be defined by raw syntax
    - ordering does matter:
        - `=c&=a&=b` does not match `=a&=b&=c`
- repeated keys with different ordering: (`a=1&a=2` vs `a=2&a=1`)
    - can be defined by json
    - can be defined by raw syntax
    - ordering does matter:
        - `a=1&a=2` does not match `a=2&a=1`
        - `a=2&a=1` does not match `a=1&a=2`
- special characters (in key and value)
    - `/`: will be encoded to `%2F`
    - `&`: will be encoded to `%26`
    - `?`: will be encoded to `%3F`
    - `=`: will be encoded to `%3D`
- only ampersands (`&&&`)
    - can't be defined by json
    - can be defined by raw syntax
    - these query strings can match it:
        - `&`
        - `&&`
        - `&&&`
        - ...
    - empty string can't match it
- only equals signs (`===`)
    - can't be defined by json because all of these cases are detected as `text/plain` by pact:
        - `{"==":""}`
        - `{"=":"="}`
        - `{"":"=="}`
    - can be defined by raw syntax
    - these query strings can match it:
        - `===`
        - `=%3D%3D`
    - empty string can't match it
- no key and value (`=&=&=`)
    - can be defined by json
    - can be defined by raw syntax
    - these query strings can't match it:
        - ``
        - `=&`
        - `=&=`
        - `=&=&`
        - `=&=&=&=`
        - `=&=&=&=&`
        - ...
    - these query strings can match it:
        - `=&=&=`
        - `=&=&=&`
- repeated keys with empty string value and different ordering: (`a=&a=1` vs `a=1&a=` vs `a=1` vs `a=`)
    - can be defined by json
    - can be defined by raw syntax
    - ordering and size does matter:
        - `a=&a=1` does not match `a=1&a=`
        - `a=&a=1` does not match `a=1`
        - `a=&a=1` does not match `a=`
        - ...

## Reference-level explanation

Here is the flow we need to implement in Rust core (pact-reference project):

- Pact implementation call FFI method `pactffi_with_body` with:
    - Content type: `application/x-www-form-urlencoded`
    - Body: Integration JSON format , which may include:
        - Matching rules
        - Example values
        - Generators
- Rust core will process the body:
    - If content type hint is `application/x-www-form-urlencoded` AND detected content type from body is `application/json`
        - Extract matching rules
        - Extract generators
        - Return example JSON body with example values e.g. `{ "key": "example value" }`
    - Example JSON body will be converted to example Form UrlEncoded body e.g. `key=example+value`
    - These example values (including matchers and generators) will be ignored:
        - Null
        - Boolean (true/false)
        - Object
        - Array of Arrays
        - Array of Objects
    - Register the interaction as normal with:
        - Extracted matching rules
        - Extracted generators
        - Example Form UrlEncoded body
        - Content type: `application/x-www-form-urlencoded`

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
- Postel's law need to be followed outside of the implementation of this RFC
- Due to lack of standards, some of the syntax will not be supported

## Future possibilities

### Support arrays and objects

Probably not a good idea, because there are no standard syntax, and each language handle it differently

Reference: https://blog.shalvah.me/posts/fun-stuff-representing-arrays-and-objects-in-query-strings

### Follow Postel's law

Following Postel's law, the provider may return fields that the consumer will just ignore. But if provider send extra param in body, we will got this mismatch:

```
Unexpected form post parameter 'extra' received
```

I believe this is a bug, and it happen independently with this RFC, we need to fix it in the future PRs.
