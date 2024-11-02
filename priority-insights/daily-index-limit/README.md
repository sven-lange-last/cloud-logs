[SPDX-License-Identifier: CC-BY-SA-4.0]: #
[Copyright © 2024 Sven Lange-Last]: #
# Daily Index Limit in IBM Cloud Logs Priority Insights

The **Priority insights** feature in IBM Cloud Logs parses log records into typed fields and stores them in an index to provide short latency and very fast queries. In order to protect the service, **Priority insights** limits the number of fields that can be stored in the daily index of an IBM Cloud Logs service instance.

When you reach the index field limit, new fields in log records are no longer indexed. When you use filters or queries in **Priority insights** on the **Logs** page on such fields, no log records will be returned. In such a case, you may have the impression that log records are missing.

This article explains:

* how log records are parsed into fields.
* what a field-based search is.
* what the daily index is.
* what happens when the daily index field limit is reached.
* how to determine whether the daily index field limit is reached.
* how to determine whether a field is in the daily index.
* what you can do to stay below the limit.

What is important to know: The **Store and search** feature in IBM Cloud Logs works differently than **Priority insights**. **Store and search** has no indexing limits. If you miss log records in **Priority insights** queries, use the same query in **All Logs** on the **Logs** page or submit it on the **Archive query** page. The **Store and search** feature requires that you connect an IBM Cloud Object Storage (COS) bucket to your IBM Cloud Logs service instance.

## Parsing log records into fields

Log records sent to IBM Cloud Logs can be text in many different formats: [JSON](https://www.json.org/), [logfmt](https://brandur.org/logfmt), [Extended Log File Format](https://www.w3.org/TR/WD-logfile.html), [Syslog](https://datatracker.ietf.org/doc/html/rfc5424), [Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format), [HAProxy log formats](https://www.haproxy.com/documentation/haproxy-configuration-manual/1-8r1/#8), ... or any text that does not follow a known format.

If a log record is in JSON format, its JSON members are parsed into named fields together with their contents. Example: `{ "key": "value" }` is parsed into a field named `key` with content `value`.

If a log record is NOT in JSON format or mixed with non-JSON data like a timestamp at the beginning of the log record, [parsing rules](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-log_parsing_rules) can be used in IBM Cloud Logs to transform the log record into JSON format that can be parsed. Parsing rules can also be used to extract portions of the log record into named fields.

The result of parsing rules and parsing: The log record is indexed (stored) in **Priority insights** as a set of fields and their contents.

(For completeness: When a log record is stored in **Priority insights**, it is also stored in **Store and search** if a COS bucket is connected, i.e. it is stored in two places in parallel.)

## What happens when Priority insights processes log records?

**Priority insights** keeps track of all known fields in its index. When a log record is processed, **Priority insights** checks whether the field names in the log record are already known. For an unknown field, a field mapping is added to the index that determines the field type plus other instructions how to index the field.

After unknown fields have been added to the index, field values from the log record are indexed for fast field-based and full-text search. In addition, the full log record is stored.

## Label and metadata fields

Log records in IBM Cloud Logs have label and metadata fields:

* `Application` or `Subsystem` are log record label fields. In Lucene queries, such fields are prefixed with `coralogix.`. In DataPrime queries, such fields are prefixed with `$l.`.
* `Timestamp` or `Severity` are a log record metadata fields. In Lucene queries, such fields are prefixed with `coralogix.`. In DataPrime queries, such fields are prefixed with `$m.`.

Label and metadata fields are received by IBM Cloud Logs together with the log record data or are populated by IBM Cloud Logs. All label and metadata fields are known fields in the **Priority insights** index. IBM Cloud Logs users can not add label and metadata fields to the index or remove them.

See following pages in IBM Cloud Logs documentation:

* [Metadata fields](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-metadata)
* [Sending logs by using the API](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-send-logs-api)
* [DataPrime reference: Expressions](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-dataprime-ref#expressions)

## Full-text and field-based search

If you don't restrict queries in **Priority insights** to a field, the search will be performed on all indexed log record fields:

* Lucene: Don't prefix a query term with `<field>:` to perform a search on all indexed log record fields - including log record label and metadata fields.
* DataPrime: Use a query like `source logs | filter $d ~~ 'stored'` to perform a search on all indexed log record **data** fields. Searching log record data, label, and metadata fields at the same time requires a more complex query.

You can use field-based filters and queries in **Priority insights** to search indexed log record fields:

1. Filters: On the **Filter** pane, you can select field values from a list of detected values. Only log records that match the selected field values will be returned. By default, the `Application`, `Subsystem`, and `Severity` fields are available for filtering. Other fields detected in log records can be added to define filters.
2. Queries: Lucene and DataPrime queries can be used to search only specified indexed fields.
   * Lucene: Use the `<field>:<query term>` syntax to apply the specified query term only to the specified field.
   * DataPrime: Use field accessors `$d.<data field>`, `$l.<label field`, or `$m.<metadata field>` to work on the specified field.

See following pages in IBM Cloud Logs documentation:

* [Filtering log data](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-query-data-filter)
* [Querying data by using Lucene](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-query-data-lucene)
* [DataPrime reference](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-dataprime-ref)

## Daily indices

Each IBM Cloud Logs service instance has a separate set of **Priority insights** indices. The first index in the set is created when the service instance is created. Once a day, a new index is created and added to the set. At the same time, the oldest index is removed if it has reached its expiration date.

A new daily index only contains the known label and metadata fields. Every day, all other fields in the index are learnt anew from the processed log records. As a result, fields in daily indices can differ if processed log records differ.

## Daily index limits

In order to protect the service, **Priority insights** limits the number of fields that can be stored in the daily index of an IBM Cloud Logs service instance.

Once the daily index field limit is reached, the following will happen when **Priority insights** processes log records:

* Unknown fields are no longer added to the daily index and their field values are not indexed. Such fields and their values are not lost because **Priority insights** always stores the full log record.
  Such fields will NOT be searched - neither in full-text nor in field-based search.
* Field values of fields that are already known in the daily index will be indexed as usual.

Example:

* **Priority insights** processes a new log record with two fields: `{ "known": "indexed", "unknown": "stored" }`.
* Field `known` already has a field mapping in the daily index. It is known. Its value `indexed` is indexed.
* Field `unknown` has no field mapping in the daily index yet. It is unknown. **Priority insights** cannot add a field mapping because the daily index limit is reached. Its value `stored` is not indexed.
* **Priority insights**  stores the full log record.
* Following queries will return the log record:
  * Lucene query `indexed`.
  * Lucene query `known:indexed`.
  * DataPrime query `source logs | filter $d ~~ 'indexed'`.
  * DataPrime query `source logs | filter $d.known ~ 'indexed'`.
* Following queries will NOT return the log record because field `unknown` has no field mapping in the daily index and value `stored` is not indexed for field `unknown`:
  * Lucene query `stored`.
  * Lucene query `unknown:stored`.
  * DataPrime query `source logs | filter $d ~~ 'stored'`.
  * DataPrime query `source logs | filter $d.unknown ~ 'stored'`.

As mentioned, a new daily index automatically contains the known label and metadata fields. They will be counted towards the daily index limit.

Since new fields are added to the index in the order in which they appear and no more fields are added after the limit is reached, the set of fields in the index and thus, the indexed field values, can differ day by day. For this reason, searches for a field may work one day but may not work the next.

## Check daily index usage

You can check the usage of the current daily index: Open the **Usage** - **Mapping stats** page. Check the `Used keys today` statistic. If all keys are used, the daily index field limit is reached.

You can only check usage for the current day, not for previous days. Typically, this statistic is meaningful unless you check immediately after the daily index is created or your log records contain many new fields shortly before the current daily index closes.

## Determine whether a field is in the daily index

On the **Explore Logs** - **Logs** page, select **Priority Insights** and the **Logs** tab. Open **Settings** on the result list header. Select to show mapping errors under `Annotations`. With that annotation option, all log record fields of the selected log record in the result list that have no mapping in the daily **Priority insights** index will be marked with a red exclamation mark indicator symbol. If you hover the mouse pointer over the indicator symbol, IBM Cloud Logs will display a message.

There are two main reasons why a field is not indexed:

1. You reached the daily index field limit of your IBM Cloud Logs service instance.
2. The field is only contained in log records that have a mapping exception. This is a different condition not discussed in this article.

## Stay below daily index field limit

There are different strategies to stay below the daily index field limit:

1. Use [TCO policies](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-tco-optimizer) to only store high priority log records in **Priority insights**.
2. Use [Stringify JSON field](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-convert-to-json-string) parsing rules to turn the value of complex JSON object values into escaped JSON.
3. Use [Remove Fields](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-remove-rule) parsing rules to remove fields from log records.
4. Use [TCO policies](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-tco-optimizer) or [Block](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-block-rule) parsing rules to prevent that log records are stored.

### TCO policies

[TCO policies](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-tco-optimizer) in IBM Cloud Logs can be used to classify received log records into different priority classes. Only logs records with high priority are processed and stored by **Priority insights**. Only logs records that are searched frequently should go into **Priority insights**.

Reducing the amount of logs in **Priority insights** will typically also reduce the number of indexed fields.

### Stringify JSON field

Use a [Stringify JSON field](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-convert-to-json-string) parsing rule to rename a field with a complex JSON object value and turn the value into escaped JSON. **Priority insights** only parses values in JSON format into named fields with content but not escaped JSON. With the parsing rule, the transformed field is no longer an `object` field but a `text` field. **Priority insights** will no longer parse the former complex JSON object value into nested fields. Content of the transformed field won't work any longer with filters or field-based queries but can still be searched with full-text queries.

This approach is useful for complex log records in JSON format where most of the nested fields will rarely be used in filters or field searches.

Example: IBM Cloud Activity Tracker [events](https://cloud.ibm.com/docs/atracker?topic=atracker-event) have [requestData](https://cloud.ibm.com/docs/atracker?topic=atracker-event#requestData) and [responseData](https://cloud.ibm.com/docs/atracker?topic=atracker-event#responseData) fields that have JSON object values. If you really need audit events in **Priority insights** instead of **Analyze and alert** or **Store and search**, you can define [Stringify JSON field](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-convert-to-json-string) parsing rules to turn the values of `requestData` and `responseData` fields into escaped JSON.

### Remove fields

Use a [Remove Fields](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-remove-rule) parsing rules to permanently delete unnecessary fields from log records. Removed fields won't be stored in **Priority insights** or **Store and search** and can no longer be used in alert conditions. There is no way to recover removed fields.

### Block log records

[TCO policies](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-tco-optimizer) or [Block](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-block-rule) parsing rules can be used permanently prevent log records from being processed and stored by **Priority insights**. There is no way to recover blocked logs.

Reducing the amount of logs in **Priority insights** will typically also reduce the number of indexed fields.

For completeness: By default, blocked logs won't be processed or stored by **Analyze and alert** and **Store and search** either. [Block](https://cloud.ibm.com/docs/cloud-logs?topic=cloud-logs-parse-block-rule) parsing rules offer an option to partially block log records and still process and store then in **Store and search**.
