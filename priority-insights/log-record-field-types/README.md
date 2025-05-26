[SPDX-License-Identifier: CC-BY-SA-4.0]: #
[Copyright © 2024, 2025 Sven Lange-Last]: #
# Log Record Field Types in IBM Cloud Logs Priority Insights

The **Priority insights** feature in IBM Cloud Logs parses log records into typed fields. By and large, the field types are not declared but dynamically inferred from field names and values when a field is processed for the first time. The advantage of typed fields is that specialized search operations based on the type can be supported.

Unfortunately, typed fields cause mapping exceptions when a log record field has a value that does not match the type inferred before. Such field mapping exceptions and how to deal with them is a complex topic of its own. A separate article explains them in more detail.

This article explains:

* how **Priority insights** detects new fields and determines their type.
* why **Priority insights** has different field types.
* what multifields are.
* which field types ignore malformed values.
* how arrays, `null` values, and objects are processed.
* how nested objects and dots (`.`) in field names work and what the resulting problems are.
* how the daily index works.
* what field mapping exceptions are.

Another article provides a set of log records with all field types introduced in this article and instructions how to send them to your IBM Cloud Logs service instance and how to query the current field mapping.

The **Priority insights** feature is based on the [OpenSearch Project](https://opensearch.org/). If you already know OpenSearch, the concepts will sound familiar to you.

What is important to know: The **Store and search** feature in IBM Cloud Logs works differently than **Priority insights**. **Store and search** uses dynamic typing of fields and converts field value types as needed. The **Store and search** feature requires that you connect an IBM Cloud Object Storage (COS) bucket to your IBM Cloud Logs service instance.

## Parsing log records into fields

Log records sent to IBM Cloud Logs can be text in many different formats: [JSON](https://www.json.org/), [logfmt](https://brandur.org/logfmt), [Extended Log File Format](https://www.w3.org/TR/WD-logfile.html), [Syslog](https://datatracker.ietf.org/doc/html/rfc5424), [Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format), [HAProxy log formats](https://www.haproxy.com/documentation/haproxy-configuration-manual/1-8r1/#8), ... or any text that does not follow a known format.

If a log record is in JSON format, its JSON members are parsed into named fields together with their contents. Example: `{ "key": "value" }` is parsed into a field named `key` with content `value`.

If a log record is NOT in JSON format or mixed with non-JSON data like a timestamp at the beginning of the log record, [parsing rules](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-log_parsing_rules) can be used in IBM Cloud Logs to transform the log record into JSON format that can be parsed. Parsing rules can also be used to extract portions of the log record into named fields.

The result of parsing rules and parsing: The log record is indexed (stored) in **Priority insights** as a set of fields and their contents.

(For completeness: When a log record is stored in **Priority insights**, it is also stored in **Store and search** if a COS bucket is connected, i.e. it is stored in two places in parallel.)

## What happens when a new field is detected?

**Priority insights** keeps track of all known fields in its index. When a log record is processed, **Priority insights** checks whether the field names are already known. For an unknown field, a field mapping is added to the index that determines the field type plus other instructions how to index the field.

After unknown fields have been added to the index, field values from the log record are indexed for fast field-based and full-text search. In addition, the full log record is stored.

## How is the field type inferred?

The field mapping depends on the field **value** and the field **name**. You may wonder why it depends on the field name. IBM Cloud Logs allows users to select some special field types based on the field name suffix because the field value would not be sufficient to safely detect the desired type. Example: If a field name ends with `_ipaddr`, its type is IP address. Without this name suffix, IP address values are classified as type text.

The table below shows the different field mappings in **Priority insights** at the time of writing this article. When a new field is detected, **Priority insights** evaluates field mapping matching conditions from top to bottom. The first field mapping that matches the new field is added to the index.

Rank | Field value has type | Field name ends with | Resulting field type | Multifields | Example values
--- | --- | --- | --- | --- | ---
1 | Any | `_geopoint` | [`geo_point`](https://opensearch.org/docs/latest/field-types/supported-field-types/geo-point/) | `keyword` with type [`keyword`](https://opensearch.org/docs/latest/field-types/supported-field-types/keyword/) | `"location_geopoint": { "lat": 48.666234, "lon": 9.0366981 }`
2 | `object` | Any | [`object`](https://opensearch.org/docs/latest/field-types/supported-field-types/object/) | None | `"new": { "key": "value" }`<br/>`"new": { }`
3 | `boolean` | Any | [`boolean`](https://opensearch.org/docs/latest/field-types/supported-field-types/boolean/) | `keyword` with type [`keyword`](https://opensearch.org/docs/latest/field-types/supported-field-types/keyword/) | `"new": true`<br/>`"new": false`
4 | Any | `_ipaddr` | [`ip`](https://opensearch.org/docs/latest/field-types/supported-field-types/ip/) | `keyword` with type [`keyword`](https://opensearch.org/docs/latest/field-types/supported-field-types/keyword/) | `"server_ipaddr": "203.0.113.42"`
5 | Any | `_custom_timestamp` | [`date`](https://opensearch.org/docs/latest/field-types/supported-field-types/date/) | `numeric` with type [`scaled_float`](https://opensearch.org/docs/latest/field-types/supported-field-types/numeric/)<br/>`keyword` with type [`keyword`](https://opensearch.org/docs/latest/field-types/supported-field-types/keyword/) | `"start_custom_timestamp": "2024-09-17T16:39:30"`<br/>`"start_custom_timestamp": "1726591170"`
6 | Any | Any | [`text`](https://opensearch.org/docs/latest/field-types/supported-field-types/text/) | `numeric` with type [`scaled_float`](https://opensearch.org/docs/latest/field-types/supported-field-types/numeric/)<br/>`keyword` with type [`keyword`](https://opensearch.org/docs/latest/field-types/supported-field-types/keyword/) | `"new": "This is a text."`<br/>`"new": "true"`<br/>`"new": 42`<br/>`"new": "48.666234,9.0366981"`<br/>`"new": "203.0.113.42"`<br/>`"new": "2024-09-17T16:39:30"`

## What are multifields?

[Multifields](https://opensearch.org/docs/latest/field-types/supported-field-types/index/#multifields) are used to index the same field differently. Example: Field `"json_number_integer": 42` is stored as ...

* `text` field named `json_number_integer`. It can be queried with Lucene query `json_number_integer:"42"` and DataPrime query `source logs | filter $d.json_number_integer ~ '42'`.
* `scaled_float` field named `json_number_integer.numeric`. It can be queried with Lucene query `json_number_integer.numeric:[42 TO 42]`. DataPrime does not fully recognize the `json_number_integer.numeric` field. For this reason, DataPrime queries use type conversion instead of subfield `source logs | filter $d.json_number_integer:number == 42`.
* `keyword` field named `json_number_integer.keyword`. It can be queried with Lucene query `json_number_integer.keyword:"42"` and DataPrime query `source logs | filter $d.json_number_integer.keyword ~ '42'`.

## Why different types?

Different field types in Priority insights support different Lucene query types. Following table shows examples which Lucene query types work well for different field types.

Field type | Lucene query type | Example field | Matching example query | 
--- | --- | --- | ---
`ip` | CIDR query | `"json_ipaddr": "203.0.113.42"` | `json_ipaddr:"203.0.113.0/24"`
`date` | Range query | `"json_custom_timestamp": "1726591170"` | `json_custom_timestamp:[1726591170\|\|-1m TO 1726591170\|\|+1h]`
`text` | Term query | `"json_string": "This is a text."` | `json_string:(a text this is)`
`text` | Fuzzy term query | `"json_string": "This is a text."` | `json_string:ex~2`
`text` | Wildcard query | `"json_string": "This is a text."` | `json_string:(te* th??)`
`text` | Range query | `"json_string": "This is a text."` | `json_string:(>=tha <=thz)`
`keyword` | Phrase query | `"json_string": "This is a text."` | `json_string.keyword:"This is a text."`
`keyword` | Regular expression query | `"json_string": "This is a text."` | `json_string.keyword:/This [si]+ a.*/`
`keyword` | Range query | `"json_string": "This is a text."` | `json_string.keyword:["This in" TO "This will"]`
`scaled_float` | Term query | `"json_number_integer": 42` | `json_number_integer.numeric:42`
`scaled_float` | Range query | `"json_number_integer": 42` | `json_number_integer.numeric:[42 TO *]`

The main difference between `text` and `keyword` fields in Priority insights:

* `text` field values are analyzed and stored as tokens. When searching text fields, you can search for tokens in any order. Example: Lucene query `json_string:(a text this is)` matches a log record that contains `"json_string": "This is a text."`.
* `keyword` field values are not analyzed but only normalized to lower case and leading / trailing whitespace is removed. When searching keyword fields, you can only search exact matches. Examples: Lucene queries `json_string.keyword:"this is a text."` and `json_string.keyword:(this IS a TEXT\.)` (`.` needs to be escaped) match a log record that contains `"json_string": "This is a text."`. Lucene queries `json_string.keyword:"this is a text"` (`.` missing at the end) and `json_string.keyword:(a text\. this is)` do not match the log record.

More details can be found here:

* [OpenSearch documentation: Query string query](https://opensearch.org/docs/latest/query-dsl/full-text/query-string)
* [Apache Lucene documentation: Classic query syntax](https://lucene.apache.org/core/9_11_1/queryparser/org/apache/lucene/queryparser/classic/package-summary.html#package.description)

## Ignoring malformed values

The field mappings for types `geo_point`, `ip`, and `date` ignore malformed values. Example: The value `This is a text.` is no valid `geo_point`, `ip`, or `date` value - i.e. it is malformed for said types. When a log record contains a `geo_point`, `ip`, or `date` field with a malformed value, that log record field value won't be indexed in **Priority insights** - other fields and their values from the same log record will be indexed and the whole log record including the malformed field value will be stored. Field values that are not indexed cannot be found in any search.

Example: A log record containing `"server_ipaddr": "This is a text."` won't be returned by Lucene query `server_ipaddr:"This is a text."` or DataPrime query `source logs | filter $d.server_ipaddr == 'This is a text.'` in **Priority insights**. (With **Store and search**, i.e. when selecting **All logs** on the **Logs** page, the queries will return the log record.)

Following queries return all log records that contain ignored fields:

* Lucene: `_exists_:_ignored`
* DataPrime: `source logs | filter $d._ignored != null`

See Elasticsearch documentation for details: [ignore_malformed](https://www.elastic.co/guide/en/elasticsearch/reference/current/ignore-malformed.html) and [The antidote for index mapping exceptions: ignore_malformed](https://www.elastic.co/observability-labs/blog/antidote-index-mapping-exceptions-ignore-malformed).

## What about arrays?

In **Priority insights**, a log record field with JSON array value is stored as a field with multiple values. What is important: All values in the array must have the same type. Example: Field `"json_number_integer": [23, 42]` is stored as `text` field with two values, i.e. `23` and `42`.

There is no explicit [array](https://opensearch.org/docs/latest/field-types/supported-field-types/index/#arrays) type. When a new field is detected, a field mapping is added to the index based on value and it doesn't make a difference whether the field has a single value or multiple values in a JSON array. Example: Both, `"json_number_integer": 42` and `"json_number_integer": [23, 42]` create exactly the same field mapping.

## What about `null` values?

JSON has the `null` value which stands for the empty value. When **Priority insights** detects a new field with value `null`, an empty array (`[]`), or an array that only contain `null` values (`[ null, ... ]`), it does not add a field mapping to the index, i.e. the field stays unknown.

It is not possible to search for `null` values with Lucene queries.

See [OpenSearch documentation: Null value](https://opensearch.org/docs/latest/field-types/supported-field-types/index/#null-value).

## What about nested objects?

In **Priority insights**, a log record field with JSON object value is stored in the index as an `object` field. Similar to a JSON object, an `object` field contains an unordered set of sub-fields with values. Any JSON member in a JSON object creates a sub-field based on member's value. In queries, a sub-field is referenced by using a [path expression](https://en.wikipedia.org/wiki/Path_expression) with dots (`.`).

Example: When a log record contains `"json_object": { "json_boolean": true, "json_string": "This is a text." }`, **Priority insights** will create three fields:

1. An `object` field named `json_object`.
2. A `boolean` field named `json_boolean` as sub-field of `json_object`. In queries, this field can be referenced as `json_object.json_boolean`.
3. A `text` field named `json_string` as sub-field of `json_object`. In queries, this field can be referenced as `json_object.json_string`.

In the example, field `json_object` has two sub-fields. Future log records with the `json_object` field need NOT contain these two sub-fields. Future log records may contain none, either, or both of these sub-fields as well as different sub-fields.

In **Priority insights**, sub-fields of an `object` field can also have type `object`. As a result, `object` fields can be nested to (almost) any depth. Path expressions with dots (`.`) are used to reference nested fields in queries. Example: `"an": {"object": {"nested:" {"in": {"an": "object"} } } }`. In this example, the innermost field named `an` is a `text` field and can be referenced as `an.object.nested.in.an`. All other fields are `object` fields.

## What about dots in field names?

As mentioned above, **Priority insights** supports nested `object` fields and path expressions with dots (`.`) are used to reference nested fields in queries. But what if log records contain fields with dots? Example: `"an.object.nested.in.an": "object"`.

Imagine **Priority insights** received a log record with following content - i.e. the same nested fields in different notation (Spoiler: It wouldn't work.): `{ "an.object.nested.in.an": "object", "an": {"object": {"nested:" {"in": {"an": "object"} } } } }`. In queries, both fields would be referenced as `an.object.nested.in.an`. This is obviously a problem. 

In order to avoid said problem, **Priority insights** interpretes field names in received log records containing dots as path expressions. Such field names are expanded as nested objects when ingesting log records. Example: `"an.object.nested.in.an": "object"` is interpreted as `"an": {"object": {"nested:" {"in": {"an": "object"} } } }`. With this approach, path expressions in field names have the same meaning when ingesting and when querying.

See [Allowing dots in field names #15951](https://github.com/elastic/elasticsearch/issues/15951) for details.

(For completeness: The **Store and search** feature in IBM Cloud Logs works differently than **Priority insights**. It will NOT expand field names containing dots as nested objects.)

## Challenges with dots in field names

Log records may contain field names with dots without being path expressions describing nested objects. Example: [Kubernetes Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) or [Kubernetes Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/) contain dots (`.`). Widely used labels are `app.kubernetes.io/name`, `app.kubernetes.io/version`, `kubernetes.io/os`. Log collectors may add these labels and their values to log records when tailing container logs in a Kubernetes cluster.

When Priority insights processes a log record with field `"app.kubernetes.io/name": "test"`, it will be added to the index as `"app": { "kubernetes": { "io/name": "test" } }` - i.e. `object` fields `app` and `kubernetes` as well as `text` field `io/name`. Most users would have expected a single `text` field named `app.kubernetes.io/name`.

## Daily indices

**Priority insights** processes log records and stores them in the index as a set of fields and their contents. All fields have a field mapping in the index that determines the field type plus other instructions how to index the field.

Each IBM Cloud Logs service instance has a separate set of **Priority insights** indices. The first index in the set is created when the service instance is created. Once a day, a new index is created and added to the set. At the same time, the oldest index is removed if it has reached its expiration date.

A new daily index only contains field mappings for known label (Examples: `Application` and `Subsystem`) and metadata (Example: `Timestamp` and `Secerity`) fields. Every day, all other fields in the index are learnt anew from the processed log records. As a result, field mappings in daily indices can differ if processed log records differ.

Example:

* On day 1, log record `{ "a": "text", "b": {} }` is processed. New fields named `a` and `b` are detected. Field `a` is added to the index as `text` field. Field `b` is added to the index as `object` field.
* On day 2, a new index is started and log record `{ "a": "text", "b": "text" }` is processed. Fields `a` and `b` do not exist yet in the new daily index. They are detected as new fields. Fields `a` and `b` are added to the index as `text` fields.

In this example, field `b` was an `object` field on day 1 and a `text` field on day 2.

Queries in **Priority insights** work on the full set of indices that belong to the service instance - not only the current daily index.

The advantage of the daily index lifecycle: **Priority insights** does not keep inferred field types *forever*. If you change field types in your log records, the system will learn the new type with the next daily index.

## What are mapping exceptions?

As mentioned before, **Priority insights** parses log records into named fields. All these fields have a field mapping in the index that determines the field type plus other instructions how to index the field.

When **Priority insights** processes a log record, some of the fields in the log record will already have a field mapping in the current daily index and others may be unknown. For known fields, the existing field mapping defines the field type. It can happen that the value of a field in a new log record is not an allowed value for the type defined by the field mapping.

Example:

* The current daily index contains a field named `json_object` with type `object`.
* Priority insights processes a new log record with field `"json_object": "This is a text."`.
* Priority insights determines that the provided value `"This is a text."` is not a allowed value for an `object` field.
* Priority insights cannot index the log record with the given `json_object` field and value.

There are other conflicts that lead to field mapping exceptions.

When a field mapping exception occurs when ingesting a log record, **Priority insights** will not process and index the log record as usual. Instead, it will transform the whole log record into a single field of type `text` and index it. As a result, full text search will still work for this log record - but not field searches.

Field mapping exceptions and how to deal with them is a complex topic of its own. A separate article explains them in more detail.
