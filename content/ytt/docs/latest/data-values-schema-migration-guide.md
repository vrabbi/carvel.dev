---
title: Schema Migration Guide
toc: "false"
---

Schema documents provide a way to declare data values along with their type, and default value. Without schemas, validating the presence of data values requires additional files containing starlark assertions.

[Learn more about schema here.](ytt-schema.md)\
[Read the schema module here.](lang-ref-ytt-schema.md)

## How does a configuration author migrate a library to use schemas?

*Converting to use schemas is done one library at a time. If your root library uses a schema, but references a private library that does not have a schema, there is no conflict and ytt should behave as expected.*

To make use of the schema feature, the library must first be using the [data values feature](ytt-data-values.md).
The goal is to convert the first data values file into a schema file.

**Given** a first data values file, `values-default.yml`:
```
#@data/values
---
key1: val1
key2:
  nested: 8080
key3: [] 
```
and a second data values file, `values-production.yml`:
```
#@data/values
---
key3:
- host: registry.dev.io
  port: 8080
  transport: tcp
  insecure_disable_tls_validation: false
#@overlay/match missing_ok=True
key4: new-val5
```

Convert `values-default.yml` into a schema file by following these steps:
1. Change the top level annotation in the document to say `#@data/values-schema`, and (optaional) rename `values-default.yml` to `values-schema.yml`

`values-schema.yml`:
```
#@data/values-schema
---
key1: val1
key2:
  nested: 8080
key3: [] 
```
2. Now we have a schema that defines three data values, but `key3` defines an empty array. 
Arrays in schemas have [special behavior](lang-ref-ytt-schema.md#inferring-defaults-for-arrays), their default value is empty, but their inferred type if from the value in the schema document. Arrays must have one and only one item, so copy an example item from `values-production.yml` and generalize the values

`values-schema.yml`:
```
#@data/values-schema
---
key1: val1
key2:
  nested: 8080
key3: 
- host: ""
  port: 0
  transport: tcp
  insecure_disable_tls_validation: false
```
3. Let's not forget to find any data values that were added in subsequent data values files, thus anything added with `@overlay/match missing_ok=True` should be added to the schema:

`values-schema.yml`:
```
#@data/values-schema
---
key1: val1
key2:
  nested: 8080
key3: 
- host: ""
  port: 0
  transport: tcp
  insecure_disable_tls_validation: false
key4: new-val5
```
Now this schema, and a trimmed down `values-production.yml`:
```
#@data/values
---
key3:
- host: registry.dev.io
  port: 8080
```
will produce the same final data values as the ytt configuration before using schemas.

4. Schema has [two annotations](lang-ref-ytt-schema.md#defining-schema-explicitly) that can be used to override the inferred typing and defaults of the node it annotates.
   - If `null` or not providing a value is valid input for one of the data values, you can use the `@schema/nullable` annotation to set the default value of that node to null, while still getting type checking if a value is provided. (This annotation cannot be use on arrays.)
   - If the constraints given by the schema on a data value need to be relaxed, do so by placing `@schema/type any=True` on the node; any value of any type (as long as valid yaml) can be provided as the value of this node.
