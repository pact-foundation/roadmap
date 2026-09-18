---
name: random_array_generator
started: 2026-03-22
pr: pact-foundation/roadmap#137
tracking_issue: pact-foundation/roadmap#0000
---

## Summary

This RFC proposes the addition of a `RandomArray` generator to the Pact specification, which allows generating arrays with a random number of items (between a specified min and max) where each item can have dynamically generated values. This is the first "structure generator" in Pact, designed to work alongside existing "value generators" to provide more realistic and varied test data for consumer-driven contracts.

## Motivation

Current generators only produce single values (e.g., a random string like `qomZtbtirs`). When using array matchers, example values are typically single-item arrays, and when arrays contain multiple values, those values are always identical.

A `RandomArray` generator would allow mock servers and stub servers to return arrays with a random number of items, where each item has unique, dynamically generated values. This is particularly useful for simulating realistic data for list-based UI screens, such as user directories or product catalogs.

## Guide-level explanation

### Basic Usage

The `RandomArray` generator accepts two parameters: `min` and `max`.

- `min`: Minimum number of items in the array (default: 1)
- `max`: Maximum number of items in the array (default: 5)


Here's a complete example:

```json
{
  "consumer": { "name": "OrderConsumer" },
  "provider": { "name": "OrderProvider" },
  "interactions": [{
    "description": "create multiple orders",
    "request": {
      "method": "POST",
      "path": "/orders",
      "body": {
        "total": 24,
        "items": [{
          "name": "xxx",
          "price": 12,
          "count": 2
        }]
      },
      "generators": {
        "body": {
          "$.items": {
            "type": "RandomArray",
            "min": 2,
            "max": 4
          },
          "$.items[*].name": {
            "type": "RandomString",
            "size": 10
          },
          "$.items[*].price": {
            "type": "RandomInt",
            "min": 1,
            "max": 100
          },
          "$.items[*].count": {
            "type": "RandomInt",
            "min": 1,
            "max": 10
          }
        }
      }
    },
    "response": {
      "status": 200,
      "body": {
        "success": true,
        "created": [{
          "id": "abc",
          "status": "pending"
        }]
      },
      "generators": {
        "body": {
          "$.created": {
            "type": "RandomArray",
            "min": 1,
            "max": 3
          },
          "$.created[*].id": {
            "type": "Uuid"
          }
        }
      }
    }
  }]
}
```

### Consumer Test Behavior

During consumer tests, the mock server returns responses with arrays containing a random number of items (between `min` and `max`), where each item has uniquely generated values:

```json
{
  "success": true,
  "created": [
    { "id": "936da01f-9abd-4d9d-80c7-02af85c822a8", "status": "pending" },
    { "id": "550e8400-e29b-41d4-a716-446655440000", "status": "pending" }
  ]
}
```

### Provider Verification Behavior

During provider verification, the verifier sends requests with arrays of random sizes, where each item contains unique values:

```json
{
  "total": 24,
  "items": [
    { "name": "aBcDeFgHiJ", "price": 42, "count": 7 },
    { "name": "xYzWvUtSrQ", "price": 88, "count": 3 },
    { "name": "pOnMlKjIhG", "price": 15, "count": 9 }
  ]
}
```

### XML Example

Given the following XML template:

```xml
<items>
  <item>
    <name>xxx</name>
    <price>12</price>
  </item>
</items>
```

With a generator applied at path `$.items.item`:

```json
{
  "generators": {
    "body": {
      "$.items.item": {
        "type": "RandomArray",
        "min": 2,
        "max": 3
      }
    }
  }
}
```

The result would be:

```xml
<items>
  <item>
    <name>xxx</name>
    <price>12</price>
  </item>
  <item>
    <name>xxx</name>
    <price>12</price>
  </item>
  <item>
    <name>xxx</name>
    <price>12</price>
  </item>
</items>
```

## Reference-level explanation

### Generator Classification

Generators are classified into two categories, each with different processing priorities:

1. **Structure Generators** (priority 0): Modify the shape or structure of data
   - `RandomArray`: Expands arrays to contain a random number of items
   
2. **Data Generators** (priority 1): Generate values for existing fields
   - `RandomInt`, `RandomString`, `Uuid`, `Regex`, etc.

Structure generators are always applied before data generators to ensure correct processing order.

### Processing Order

The implementation ensures correct processing order through the following steps:

1. Filter generators by test mode (Consumer or Provider)
2. Sort generators by their processing category priority
3. Apply structure generators first
4. Apply data generators second

This ordering ensures that array expansion occurs before value generation, enabling nested generators to function correctly.

### Implementation Details

When the `RandomArray` generator is applied:

#### JSON Processing

For JSON bodies, the `RandomArray` generator:

1. Takes a template item (typically a single-item array as an example)
2. Generates a random length between `min` and `max`
3. Creates an array by duplicating the template item
4. Subsequent data generators then populate each item with unique values

#### XML Processing

For XML bodies, the `RandomArray` generator:

1. Identifies the target element based on the provided path
2. Finds matching child elements
3. Clones the first matching element to reach the desired count
4. Preserves element attributes, namespaces, and nested structure

### Corner Cases

#### `min < 1`

If `min` is less than 1, the generator returns an error:

```
RandomArray: min (0) must be >= 1
```

#### `min > max`

If `min` is greater than `max`, the generator returns an error:

```
RandomArray: min (5) must be <= max (3)
```

#### Empty Template Array

If the template array is empty, the generator returns an error:

```
RandomArray: at least 1 item is required in the array to clone
```

#### Non-Array Value

If the generator is applied to a non-array value, it returns an error:

```
can only use Array generator with arrays, got: {"name": "xxx"}
```

#### Nested Generators

Nested generators are fully supported. For example:

```json
{
  "body": {
    "orders": [{
      "items": [{ "name": "item", "price": 10 }]
    }]
  },
  "generators": {
    "body": {
      "$.orders": {
        "type": "RandomArray",
        "min": 1,
        "max": 2
      },
      "$.orders[*].items": {
        "type": "RandomArray",
        "min": 2,
        "max": 3
      },
      "$.orders[*].items[*].name": {
        "type": "RandomString",
        "size": 8
      }
    }
  }
}
```

This configuration generates 1-2 orders, each containing 2-3 items, with each item having a randomly generated 8-character name.

## Drawbacks

- New version of Pact Specification must be introduced (4.1) to include this `RandomArray` generator.

## Rationale and alternatives

### Alternative 1: Use `count` parameter in Pact-js matchers

Some matchers in Pact-js accept an optional `count` parameter to control the number of examples generated:

- `atLeastOneLike(template, count)`
- `atLeastLike(template, count)`
- `atMostLike(template, count)`
- `constrainedArrayLike(template, count)`

**Rationale for rejection**:
- Only available in pact-js, not language-agnostic
- Generated examples are always identical

### Alternative 2: XML `examples` attribute

XML provides the ability to control the number of `examples` for an XML element by defining the generator inline with the body:

```json
{
  "name": "items",
  "children": [
    {
      "pact:generator:type": "type",
      "value": {
        "name": "item",
        "children": [
          {
            "content": "Item list"
          }
        ]
      },
      "examples": 2
    }
  ]
}
```

**Rationale for rejection**:
- Only available for XML, not JSON
- Generated examples are always identical

## Unresolved questions

None

## Future possibilities

1. **Other structure generators**:
   - `RandomMap`: Generate maps with a random number of key-value pairs

2. **Other generator types** (besides structure and data generators)
