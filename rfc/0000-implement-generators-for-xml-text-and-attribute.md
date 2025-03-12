---
name: implement-generators-for-xml-text-and-attribute
started: 2024-02-10
pr: pact-foundation/roadmap#0000
tracking_issue: pact-foundation/roadmap#0000
---
## Summary

This RFC proposes implementing generators for bodies of type `application/xml`.

## Motivation

Currently if we define generators for text or attribute of XML's body, mock server will return this error:

> Generators are not supported with XML

And there will be an error message in the log:

> UNIMPLEMENTED: Generators are not currently supported with XML

To fix this error, we need to implement generators for XML.

## Guide-level explanation

### Text

To define generator for XML element's text, we can use this JSON format:

```json
{
  "name": "release-date",
  "children": [
    {
      "pact:generator:type": "Date",
      "matcher": {
        "format": "dd-MM-yyyy",
        "pact:matcher:type": "date"
      },
      "content": ""
    }
  ],
  "attributes": []
}
```

Mock server should return XML response like this:

```xml
<?xml version='1.0'?><movie><release-date>27-11-2012</release-date></movie>
```

### Attribute

To define generator for XML element's attribute, we can use this JSON format:


```json
{
  "name": "specs",
  "children": [],
  "attributes": {
    "runtime": {
      "pact:generator:type": "Regex",
      "regex": "\\d+ hours [0-5]?[0-9] minutes",
      "pact:matcher:type": "regex",
      "value": ""
    },
    "aspect-ratio": {
      "pact:generator:type": "Regex",
      "regex": "([0-9]+[.])?[0-9]+:[0-9]+",
      "pact:matcher:type": "regex",
      "value": ""
    },
    "color": {
      "pact:generator:type": "Regex",
      "regex": "color|black&white",
      "pact:matcher:type": "regex",
      "value": ""
    }
  }
}
```

Mock server should return XML response like this:

```xml
<?xml version='1.0'?><movie><specs runtime='1 hours 23 minutes' aspect-ratio='3:4' color='color'/></movie>
```

## Reference-level explanation

### Implementation

- Begin at root element
    - Apply generator for each attribute
    - Apply generator for each child element (recursively)
    - Apply generator for each child text
    - If there are no text & there is generator for text, then generate text and append to the element

### Corner cases

#### Text

- Element has no text
    - Generator: `RandomInt(0, 999)`
    - Path: `$.a['#text']`
    - Before: `<?xml version='1.0'?><a></a>`
    - After: `<?xml version='1.0'?><a>123</a>`
    - Note: New child text is append to the element
- Element has a child element
    - Generator: `RandomInt(0, 999)`
    - Path: `$.a['#text']`
    - Before: `<?xml version='1.0'?><a><b/></a>`
    - After: `<?xml version='1.0'?><a><b/>123</a>`
    - Note: New child text is append to the element
- Element has mixed children
    - Generator: `RandomInt(0, 999)`
    - Path: `$.a['#text']`
    - Before: `<?xml version='1.0'?><a><b/>-1</a>`
    - After: `<?xml version='1.0'?><a><b/>123</a>`
- Element has multiple texts
    - Generator: `RandomInt(0, 999)`
    - Path: `$.a['#text']`
    - Before: `<?xml version='1.0'?><a>-1<b/>-2</a>`
    - After: `<?xml version='1.0'?><a>123<b/>234</a>`
    - Note: We can't define index for the text (e.g. can't do this `$.a['#text'][0]` or this `$.a['#text'][1]`), the same generator is applied to every texts
- Multiple elements has text
    - Generator: `RandomInt(0, 999)`
    - Path: `$.root.a['#text']`
    - Before: `<?xml version='1.0'?><root><a>-1</a><a>-2</a></root>`
    - After: `<?xml version='1.0'?><root><a>123</a><a>234</a></root>`
- Element has namespace
    - Generator: `RandomInt(0, 999)`
    - Path: `$.n:a['#text']`
    - Before: `<?xml version='1.0'?><n:a xmlns:n='http://example.com/namespace'>-1</n:a>`
    - After: `<?xml version='1.0'?><n:a xmlns:n='http://example.com/namespace'>123</n:a>`
- Element has mixed namespace
    - Generator: `RandomInt(0, 999)`
    - Path: `$.root.n:a['#text']`
    - Before: `<?xml version='1.0'?><root><n:a xmlns:n='http://example.com/namespace'>-1</n:a><a>-2</a></root>`
    - After: `<?xml version='1.0'?><root><n:a xmlns:n='http://example.com/namespace'>123</n:a><a>-2</a></root>`


#### Attribute

- Attribute has namespace
    - Generator: `RandomInt(0, 999)`
    - Path: `$.a['@n:attr']`
    - Before: `<?xml version='1.0'?><a n:attr='-1' xmlns:n='http://example.com/namespace'/>`
    - After: `<?xml version='1.0'?><a n:attr='123' xmlns:n='http://example.com/namespace'/>`
- Attribute has mixed namespace
    - Generator: `RandomInt(0, 999)`
    - Path: `$.a['@n:attr']`
    - Before: `<?xml version='1.0'?><a n:attr='-1' attr='-2' xmlns:n='http://example.com/namespace'/>`
    - After: `<?xml version='1.0'?><a n:attr='123' attr='-2' xmlns:n='http://example.com/namespace'/>`

## Drawbacks

None

## Rationale and alternatives

None

## Unresolved questions

None

## Future possibilities

In the future, when we can define index for the text, then we can define different matcher/generator for each text, then we can apply different generator for each text:

- Generators:
    - Path: `$.a['#text'][0]`
      Generator: `RandomInt(0, 999)`
    - Path: `$.a['#text'][1]`
      Generator: `RandomString(3)`
- Before: `<?xml version='1.0'?><a>-1<b/>-2</a>`
- After: `<?xml version='1.0'?><a>123<b/>abc</a>`
