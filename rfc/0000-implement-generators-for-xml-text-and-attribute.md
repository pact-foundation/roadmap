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

- For each generator
  - Begin at root element
      - Apply generator for each attribute (if the path is matched)
      - Apply generator for each child text (if the path is matched)
      - Apply generator for each child element (recursively)
      - If the element is empty & there is generator for the text, then generate the text and append it to the element

Generators are applied to text and attribute only. Here is how the generator interact with other structures of XML:

* UTF-8 (e.g. `<俄语 լեզու="ռուսերեն">данные</俄语>`): Generator can work with tag, attribute and text that contains UTF-8 characters
* XML Declaration (e.g. `<?xml version="1.0" encoding="ASCII"?>`): Generator does not add/change XML declaration
* Empty tag (e.g. `<data />` or `<data></data>`): Generator append the text into the empty tag (if the path is matched) e.g. `<data />` or `<data></data>` become `<data>abcxyz</data>`
* White-space preserving (i.e. `xml:space="preserve"`): Generator does not add/change extra while-space except those while-space belong to attribute's value and text
* Escaping data (e.g. `<example>&lt;foo /&gt;</example>`): Generator will escape generated text and attribute's value
* Comment (e.g. `<!-- some explanation -->`): Generator does not add/change comment
* Array of element (e.g. `<data><tag>1</tag><tag>2</tag></data>`): Generator will apply to text and attribute of all elements (if the path is matched)
* Type Declaration (e.g. `<!ELEMENT data (tag*)>`): Generator does not add/change Type Declaration
* Attribute-list Declaration (e.g. `id ID #REQUIRED`): Generator does not add/change Attribute-list Declaration
* Entity Declaration (e.g. `<!ENTITY domain "github.com">`): Generator does not add/change Entity Declaration
* Namespace (e.g. `<n:a xmlns:n='http://example.com/namespace'>test</n:a>`)
  * Generator can work with element and attribute that has namespace
  * Generator can not add/change namespace for element and attribute
* Processing Instruction (e.g. `<?xml-stylesheet type="text/xsl" href="style.xsl"?><root/>`): Generator does not add/change Processing Instruction

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
- Multiple elements at different path has text
    - Generator: `RandomInt(0, 999)`
    - Path: `$.root.*.c.*['#text']`
    - Before: `<?xml version='1.0'?><root><a><c><d>-1</d><d>-2</d></c></a><b><c><e>-3</e><e>-4</e></c></b></root>`
    - After: `<?xml version='1.0'?><root><a><c><d>123</d><d>234</d></c></a><b><c><e>345</e><e>456</e></c></b></root>`
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
- Element has UTF-8 text
    - Generator: `RandomString(3)`
    - Path: `$.俄语['#text']`
    - Before: `<?xml version='1.0'?><俄语>данные</俄语>`
    - After: `<?xml version='1.0'?><俄语>abc</俄语>`
- Escaping text
    - Generator: `Regex("<foo/>")`
    - Path: `$.a['#text']`
    - Before: `<?xml version='1.0'?><a>1</a>`
    - After: `<?xml version='1.0'?><a>&lt;foo/&gt;</a>`


#### Attribute

- Multiple elements has attribute
    - Generator: `RandomInt(0, 999)`
    - Path: `$.root.a['@attr']`
    - Before: `<?xml version='1.0'?><root><a attr='-1'/><a attr='-2'/></root>`
    - After: `<?xml version='1.0'?><root><a attr='123'/><a attr='234'/></root>`
- Multiple elements at different path has text
    - Generator: `RandomInt(0, 999)`
    - Path: `$.root.*.c.*['@attr']`
    - Before: `<?xml version='1.0'?><root><a><c><d attr='-1'/><d attr='-2'/></c></a><b><c><e attr='-3'/><e attr='-4'/></c></b></root>`
    - After: `<?xml version='1.0'?><root><a><c><d attr='123'/><d attr='234'/></c></a><b><c><e attr='345'/><e attr='456'/></c></b></root>`
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
- Element has UTF-8 attribute
    - Generator: `RandomString(3)`
    - Path: `$.俄语['@attr']`
    - Before: `<?xml version='1.0'?><俄语 լեզու="ռուսերեն"/>`
    - After: `<?xml version='1.0'?><俄语 լեզու="abc"/>`
- Escaping attribute
    - Generator: `Regex("' new-attr='\"val")`
    - Path: `$.a['@attr']`
    - Before: `<?xml version='1.0'?><a attr='1'/>`
    - After: `<?xml version='1.0'?><a attr='&apos; new-attr=&apos;&quot;val'/>`

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
