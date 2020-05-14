# JSON-Schema logical operators and JSON Merge

This document describes various uses of JSON-schemas logical operators, anyOf, allOf, and oneOf and how they can be used to constrain input objects. The first section describes and ideal table that holds all named references, and how anyOf, allOf, oneOf might apply to this system.

Object composition can also be performed with JSON-merge, and it may be desirable to not only templatize a particular output, but templatize particular json objects and compose them with json merge to produce a final output.

# Exploration

I assume I have a schema table with all the schema I know about in the Set
this should hold a reference to every named object we know about, it's
schema, and links to dependent schemas ($refs)
some objects will be a link to another hash in the table (pure $ref)
this table should be a hash table and not a schema!

schema is used here for illustrative purposes only, the hash table described
above would be populated when the templateSet is loaded

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



# oneOf

exclusive choice (exclusive or)
only one schema is rendered, based on the passing schema
ambiguous parameters objects (multiple schemas pass) are errors
if no schema passes, this is a more traditional validation error
```javascript
const choice_schema = {
  oneOf: [
    { '$ref': 'schema_table#/definitions/counting_number'},
    { '$ref': 'schema_table#/definitions/conditional'},
    { '$ref': 'schema_table#/definitions/word'}
  ]
}
```
in practice, if the sets are known to be mutually exclusive, the matching
reference type can be used to provide a form of reflection to the caller:
based on which schema validates, I could choose a behavior.

validation behavior could provide 'nand' to remove a set from a larger set
```javascript
const remove_subset_schema = {
  oneOf: [
    { '$ref': 'schema_table#/definitions/counting_number'},
    { '$ref': 'schema_table#/definitions/selected_primes'},
  ]
}
```

this filters out selected primes, this is one way to implement subtraction
from a set
no reflection necessary because only a subset of a type is allowed


validation here provides an exclusive or, not both, not niether

```javascript
const not_intersection_schema = {
  oneOf: [
    { '$ref': 'schema_table#/definitions/selected_primes'},
    { '$ref': 'schema_table#/definitions/selected_fibonacci'},
  ]
}
```

this excludes the intersection of these sets, 2, 3, 5, and 13 from passing

### in summary

- planned mutual exclusivity of sets is a plus
- provide choice and match
- if sets are not mutually exclusive, oneOf may have surprising but powerful features

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
    { '$ref': 'schema_table#/definitions/selected_prime_strings' },
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


- merged templates must all have the same contentMediaType
- merge behavior defined per contentMediaType
- unsupported contentMediaTypes will append the template text in the order they show up in the allOf/anyOf ... merge does not apply to oneOf
- default behavior is contentMediaType: 'application/json', 'merge'
- would like to expand to application/yaml
- template set should not validate/build/load if conflicting types are used 

```javascript
const _ = require('lodash');
```
