# JSON-Schema logical operators and JSON Merge

JSON-schema is an invaludable tool for providing input validation to rest APIs and other JavaScript processes. There are various ways of authoring schema and combining schema, as well as referencing external schema. A single self contained JSON-schema is easy enough to parse and follow, as its elements are all presented in a single block. However, parsing a *Schema Set* provided in many different files presents another challenge. Many root objects may be presented, the top elements of the Schema Set, and they may reference each other and be deeply interrelated.

The FAST template system adds an additional *template* property to JSON-schema. A Schema Set with root objects that contain the template property are called *FAST Templates* or merely *Templates*. Validation of a Template ensures that all template parameters (or tags) have an associated schema, and that all `$ref`s can be followed. Validation of a *Template Set* ensures this across a fileset. A Template Set can then produce a Schema Set.

Writing FAST templates is a technique for JSON-schema authoring, and the primary output of a rendered FAST template is a JSON Object. From here on, the discussion will remain focused on the relationship between JSON-schema and JSON Objects.

Discussed in detail are *validation sets*. A *validation set* for a particular schema is the set of all JSON objects that pass validation using that schema. If a JSON object passes Schema validation, it is a member of that schema's validation set. Every schema describes a validation set (also then note, each schema set has a set of associated validation sets). Schemas that describe the same validation set are said to be equivalent.

# Equivalent Schema

Schema can contain many extraneous properties and structures that result in the same validation set. Consider the following:

```json
{
  "type": "object",
  "properties": {
    "key": {
      "type": "string",
      "regex": "[a-Z]*"
    },
    "value": {
      "type": "string"
    }
  }
}
```
and:
```json
{
  "type": "object",
  "definitions": {
    "key": {
      "type": "string",
      "regex": "[a-Z]*"
    },
    "value": {
      "type": "string"
    }
  },
  "allOf": [
    { "$ref": "#/definitions/key" },
    { "$ref": "#/definitions/value" }    
  ]
}
```

Both schemas describe the same validation set. One schema uses the properties keyword to define object properties within a particular scope, and the other schema uses allOf to combine definitions to mix child namespaces.

To further confuse things, these approaches can be mixed to yield a third, equivalent representation:

```json
{
  "type": "object",
  "properties": {
    "key": {
      "type": "string",
      "regex": "[a-Z]*"
    }    
  },
  "definitions": {
    "value": {
      "type": "string"
    }
  },
  "allOf": [
    { "$ref": "#/definitions/value" }      
  ]
}
```

We can do this for days, note the allOf operator can be applied ad infinitum, ad nauseum:

```json
{
  "allOf": [{
    "allOf": [{
      "allOf": [{
        "allOf": [
          { "type":"object" }
        ]
      }]
    }]
  }]
}
```
No matter how many allOfs, oneOfs, anyOfs in the chain, it all reduces to:
```json
{ "type" : "object "}
```

This observation produces a rule:
```javascript
{ "allOf": [ {} ] } === {}
```
Applying this rule to the former schema four times produces a more elegant representation of the validation set. Is this a minimal representation of the validation set?

Using `$ref` we can discover a similar rule:

```javascript
{
  "definitions" : {
    "schemaN": { "$ref": "#/definitions/schemaN-1" },
      ...
    "schema1": { "$ref": "#/definitions/schema0" },
    "schema0": {}  
  },
  "$ref" : "#/definitions/schemaN"  
} === {}
```

No matter how many naked references we follow, the schema is unaltered.

We have established that many schema representations provide equivalent validation sets, and that rules exist to remove no-ops from the schema representation. Can a canonical schema be defined for a given validation set?

*FAST should strive to provide a single sensible schema representation for a given template specification, not necessarily a canonical schema*

# Using Logic to Combine Schema

The allOf, oneOf, and anyOf operators can be used to manipulate schema by applying some generic logical operations to the schema. 'oneOf' acts like xor, 'allOf' acts like and, and 'anyOf' acts like or. This can give us a sort of intuition for how schema should operate.

# oneOf

exclusive choice (exclusive or)
- only one schema is rendered, based on the passing schema
- ambiguous parameters objects (i.e. multiple passing schemas) are errors
- if no schema passes, this is a validation error

```javascript
const choice_schema = {
  oneOf: [
    { '$ref': 'schema_table#/definitions/counting_number'},
    { '$ref': 'schema_table#/definitions/conditional'},
    { '$ref': 'schema_table#/definitions/word'}
  ]
}
```

We can use any schemas we'd like in the one of, however if two schemas have overlapping sets, the overlap will be disallowed.

In practice, the matching reference type can be used to provide a form of reflection to the caller: based on which schema validates, I can choose a behavior. oneOf provides a guard against ambiguous schema, this can be frustating for the user if they recieve multiple validation errors for a particular input. It may not be clear how to change the input to make it pass only one schema.

Making the schemas mutually exclusive can help. Using a [type discriminator](https://swagger.io/docs/specification/data-models/inheritance-and-polymorphism/) (such as a `class` property), the most relevant error message can be passed to the user and ensures the schemas are mutually exclusive. If there is no discriminator value that matches to a discriminator in oneOf, the user can be provided a different error message that enumerates acceptable discriminator values.

The next example shows a oneOf with overlapping validation sets.


```javascript
const remove_subset_schema = {
  oneOf: [
    { '$ref': 'schema_table#/definitions/counting_number'},
    { '$ref': 'schema_table#/definitions/selected_primes'},
  ]
}
```

This filters out selected primes, this is one way to implement subtraction
from a set. Reflection is not useful in this particular example, because only a subset of a type is allowed.

Validation here provides an exclusive or, not both/all, just one. Applying this operation in chains, using intersecting schemas, will remove elements from the validation set.  


```javascript
const not_intersection_schema = {
  oneOf: [
    { '$ref': 'schema_table#/definitions/selected_primes'},
    { '$ref': 'schema_table#/definitions/selected_fibonacci'},
  ]
}
```
this excludes the intersection of these sets, 2, 3, 5, and 13 from passing, but it adds 7, 11, 17, 19, 131 and 1, 8, 21 into an unambiguous validation set. If there are fuzzy rules for processing ambiguous inputs, in this case 2, 3, 5, and 13, the input must be passed to another system for complex handling.  

### in summary

- planned mutual exclusivity of sets is a plus, more predictable
- a discriminator can be used to provide choice, and match behavior
- if sets are not mutually exclusive, oneOf subtracts intersections from the provided validation sets.


# anyOf

multiple choice, non exclusive (logical or)
any number of choices can be included
when rendering templates -- templates that do not pass the schema would not be rendered
```javascript
const optional_schema = {
  anyOf: [
    { type: 'number' },
    { type: 'boolean' },
    { type: 'string' }
  ]
}
```
in practice, this can be used to create validation on unions of sets

anyOf is fairly forgiving... bringing sets of inputs together
may be easy to unintentionally allow bad objects in complex schemas

# allOf

*impossible!*
```javascript
const impossible_required_schema = {
  allOf: [
    { type: 'number' },
    { type: 'boolean' },
    { type: 'string' }
  ]
}
```
"all of" schemas must share the same json-schema type

must pass all these schemas (logical and)
```javascript
const possible_required_schema = {
  allOf: [
    { '$ref': 'schema_table#/definitions/selected_fibonacci' },
    { '$ref': 'schema_table#/definitions/selected_prime_strings' }
  ]
}
```
allows 2, 3, 5, and 13. Intersection of sets.
Inverse of set defined by not_intersection_schema

allOf in a template is a merge operation!
if you tried to validate the template, you'd need all the variables
in the templates, with the required ones there
the generated schema is easy to create, because it follows naturally from
use of allOf in a template

since the schema passes allOf the template schemas, we can use it to render
individual templates. we have two options;

parse the rendered text using its content type, and merge the parsed results
according to an appropriate strategy to the content type

append all templates in the order they appear in the allOf





Object composition can also be performed with JSON-merge, and it may be desirable to not only templatize a particular output, but templatize particular json objects and compose them with json merge to produce a final output.

This document describes various uses of JSON-schemas logical operators, anyOf, allOf, and oneOf. These operators can be extremely powerful for describing input spaces, and constraints or additions to those input spaces. An input space is simply the set of all inputs a program or function will accept.

# Exploration

I assume I have a schema table with all the schema I know about in the Set
this should hold a reference to every named object we know about, it's
schema, and links to dependent schemas ($refs)
some objects will be a link to another hash in the table (pure $ref)

In memory, this table should be a hash table and not a schema. The templateSet shares a single definitions namespace. Some definitions will need anonymous hashes. schema is used here for illustrative purposes only, the hash table described above would be populated when the templateSet is loaded

```javascript
const schema_table = {
  id: 'schema_table',
  definitions: {
    counting_number: {
      type: 'integer'
    },
    conditional: {
      type: 'boolean'
    },
    letters: {
      type: 'string'
    },
    symbol: {
      type: 'string',
      regex: '[a-Z0-9]*'
    },
    digits: {
      type: 'string',
      regex: '[0-9]*'
    },
    word: {
      type: 'string',
      regex: '[a-Z]*'
    },
    important_words: {
      type: 'string',
      enum: [ 'food', 'water', 'sleep' ]
    },
    selected_primes: {
      type: 'integer',
      enum: [ 2, 3, 5, 7, 11, 13, 17, 19, 131 ]
    },
    selected_fibonacci: {
      type: 'integer',
      enum: [ 1, 2, 3, 5, 8, 13, 21 ]
    }
  }
}
```




- merged templates must all have the same contentMediaType
- merge behavior defined per contentMediaType
- unsupported contentMediaTypes will append the rendered text in the order the templates show up in the allOf/anyOf ... merge does not apply to oneOf
- default behavior is contentMediaType: 'application/json', 'merge'
- preserving order in this pattern can map nicely to javscript prototypical inheritance
- would like to expand to application/yaml
- template set should not validate/build/load if conflicting types are used
- array merger behavior should be defined by enum: [ concat, union, merge ]

```javascript
const _ = require('lodash');
```
