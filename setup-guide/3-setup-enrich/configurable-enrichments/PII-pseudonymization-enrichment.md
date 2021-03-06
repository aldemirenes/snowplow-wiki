<a name="top" />

[**HOME**](Home) » [**SNOWPLOW SETUP GUIDE**](Setting-up-Snowplow) » [Step 3: Setting up Enrich](Setting-up-enrich) » [Configurable enrichments](Configurable-enrichments) » PII Pseudonymization Enrichment

You can find documentation for version 1-0-0 of the config for the enrichment in
[the dedicated wiki page](PII-pseudonymization-enrichment-1-0-0), which is used in pre r106 versions.

### Compatibility

JSON Schema   [iglu:com.snowplowanalytics.snowplow.enrichments/pii_enrichment_config/jsonschema/2-0-0][schema]<br/>
Compatibility R106 <br/>
Data provider None (Internal transformation)<br/>

#### Please note:
The PII Enrichment is not compatible with older versions of [Snowplow Mini](https://github.com/snowplow/snowplow-mini/wiki/Usage-guide). Only Snowplow Mini 0.5.0 and higher support the PII enrichment. 

### Overview

The PII Enrichment provides capabilities for Snowplow operators to better protect the privacy rights of data subjects. The obligations of handlers of Personally Identifiable Information (PII) data under GDPR have been outlined on the [EU GDPR website][gdpr].

The r106 release adds the capability to emit a stream of events to re-identify the data subject.
As with r100, you can configure the enrichment to pseudonymize any of the following datapoints:

1. Any of the “first-class” fields which are part of the Canonical event model, are scalar fields containing a single string and have been identified as being potentially sensitive (see **`pii_enrichement_config` JSON schema** (linked above) for full list).
2. Any of the properties within the JSON instance of a Snowplow self-describing event or context (wherever that context originated). You simply specify the Iglu schema to target and a JSON Path to identify the property or properties within to pseudonomize

In addition, you must specify the “strategy” that will be used in the pseudonymization. Currently the available strategies involve hashing the PII, using one of the following algorithms:

* `MD2`, the 128-bit algorithm [MD2][md2] (not-recommended due to performance reasons see [RFC6149][rfc6149])
* `MD5`, the 128-bit algorithm [MD5][md5]
* `SHA-1`, the 160-bit algorithm [SHA-1][sha-1]
* `SHA-256`, 256-bit variant of the [SHA-2][sha-2] algorithm
* `SHA-384`, 384-bit variant of the [SHA-2][sha-2] algorithm
* `SHA-512`, 512-bit variant of the [SHA-2][sha-2] algorithm

Additional configurations are:
1. Whether to emit re-identification events
2. The salt to use in hashing. **It is important to note that changing this will change the hash for all original values**, for instance the value '52.91.x.x' in user IP address with the value 'pink123' as salt would hash to '1f25c6b95380e6f125f9c2bf2e9b9553fb2bd8d9' (using `SHA-1`), but it would hash to '0df86a4e96d7239d2b617bddb938ee4e0ebac556' using 'pepper123' as the salt. As a consequence you may not be able to join on those fields and/or create duplicate values.

### Prerequisite checks

In configuring the PII enrichment you will need to check that all fields comply with the following two checks:

#### Format
Always check the underlying JSON Schema to avoid accidentally invalidating an entire event using the PII Enrichment. Specifically, you should check the field definitions of the fields for any constraints that hold under plaintext but not when the field is hashed, such as field length and format.

The scenario to avoid is as follows:

* You have a `customerEmail` property in a JSON Schema which must validate with <br/>`format: email`
* You apply the PII Enrichment to hash that field
* The enriched event *is* successfully emitted from Stream Enrich...
* **However**, a downstream process (e.g. RDB Shredder) which validates the now-pseudonymized event will **reject** the event, as the hashed value is no longer in an email format

#### Length

The same issue can happen with properties with enforced string lengths - note that all of the currently supported pseudonymization functions will generate hashes of up to **128 characters** (in the case of SHA-512); be careful if the JSON Schema enforces a shorter length, as again the event will fail downstream validation. Every hashed field will be a hex string, which means that they will be 1/4 of the output bit length of the algorithm (1 hexadecimal digit = 0-f = 0-15 decimal = 0000-1111 binary => 4 bits)

### Example

Here is an example configuration:

```json
{
  "schema": "iglu:com.snowplowanalytics.snowplow.enrichments/pii_enrichment_config/jsonschema/2-0-0",
  "data": {
    "vendor": "com.snowplowanalytics.snowplow.enrichments",
    "name": "pii_enrichment_config",
    "emitEvent": true,
    "enabled": true,
    "parameters": {
      "pii": [
        {
          "pojo": {
            "field": "user_id"
          }
        },
        {
          "pojo": {
            "field": "user_fingerprint"
          }
        },
        {
          "json": {
            "field": "unstruct_event",
            "schemaCriterion": "iglu:com.mailchimp/subscribe/jsonschema/1-*-*",
            "jsonPath": "$.data.['email', 'ip_opt']"
          }
        }
      ],
      "strategy": {
        "pseudonymize": {
          "hashFunction": "SHA-1",
          "salt": "pepper123"
        }
      }
    }
  }
}
```

The configuration above is for a Snowplow pipeline that is receiving events from the Snowplow JavaScript Tracker, plus a Mailchimp webhook integration:

* The Snowplow JavaScript Tracker has been configured to emit events which includes the `user_id` and `user_fingerprint` fields
* The Mailchimp webhook (available since [release 0.9.11][release-0.9.11]) is emitting `subscribe` events (among other events, ignored for the purpose of this example)

With the above PII Enrichment configuration, then, you are specifying that:

* You wish for the `user_id` and `user_fingerprint` from the Snowplow Canonical event model fields to be hashed (the full list of supported fields for pseudonymization is viewable [in the enrichment configuration schema][pii-config-schema])
* You wish for the `data.email` and `data.ip_opt` fields from the Mailchimp `subscribe` event to be hashed, but only if the schema version begins with `1-`
* You wish to use the `SHA-256` variant of the algorithm for the pseudonymization
* You wish for the re-identification events to be emitted to the pii stream (see [stream enrich configuration][stream-enrich-configuration] for configuring the stream)
* You wish for the salt value `pepper123` to be used in hashing all the values

### Data sources

There is no external input to this enrichment.

### Data generated

The emitted event is a standard *Enriched Event* formatted event (i.e. TSV) as described in [Canonical event model][canonical-event-model] so that it can be used with the [snowplow analytics SDK][analytics-sdk]. For the implementation details please look at the source code [here][stream-enrich-pii-emit], however there are a couple of fields of particular interest, namely the `contexts` and `unstruct_event`:

Firstly, the `contexts` field in this new event contains a new event type called `parent_event` with a new [schema][parent-event-schema]. Here is an example of such an event:
```json
{
  "schema": "iglu:com.snowplowanalytics.snowplow/contexts/jsonschema/1-0-0",
  "data": [
    {
      "schema": "com.snowplowanalytics.snowplow/parent_event/jsonschema/1-0-0",
      "data": {
        "parentEventId": "a0f0213e-d514-44e5-8c3d-b1fba8c54f0f"
      }
    }
  ]
}
```
This event simply contains the UUID of the parent event where the PII enrichment was applied, giving one the opportunity to verify the enrichment.

The second field of note is the `unstruct_event` field that follows the new [pii-transformation event schema][pii-transformation-schema]. An instance of that event could look like this (depending on the fields configured):
```json

{
  "schema": "iglu:com.snowplowanalytics.snowplow/unstruct_event/jsonschema/1-0-0",
  "data": {
    "schema": "iglu:com.snowplowanalytics.snowplow/pii_transformation/jsonschema/1-0-0",
    "data": {
      "pii": {
        "pojo": [
          {
            "fieldName": "user_fingerprint",
            "originalValue": "its_you_again!",
            "modifiedValue": "27abac60dff12792c6088b8d00ce7f25c86b396b8c3740480cd18e21068ecff4"
          },
          {
            "fieldName": "user_ipaddress",
            "originalValue": "70.46.123.145",
            "modifiedValue": "dd9720903c89ae891ed5c74bb7a9f2f90f6487927ac99afe73b096ad0287f3f5"
          },
          {
            "fieldName": "user_id",
            "originalValue": "john@acme.com",
            "modifiedValue": "7d8a4beae5bc9d314600667d2f410918f9af265017a6ade99f60a9c8f3aac6e9"
          }
        ],
        "json": [
          {
            "fieldName": "unstruct_event",
            "originalValue": "50.56.129.169",
            "modifiedValue": "269c433d0cc00395e3bc5fe7f06c5ad822096a38bec2d8a005367b52c0dfb428",
            "jsonPath": "$.ip",
            "schema": "iglu:com.mailgun/message_clicked/jsonschema/1-0-0"
          },
          {
            "fieldName": "contexts",
            "originalValue": "bob@acme.com",
            "modifiedValue": "1c6660411341411d5431669699149283d10e070224be4339d52bbc4b007e78c5",
            "jsonPath": "$.data.emailAddress2",
            "schema": "iglu:com.acme/email_sent/jsonschema/1-1-0"
          },
          {
            "fieldName": "contexts",
            "originalValue": "jim@acme.com",
            "modifiedValue": "72f323d5359eabefc69836369e4cabc6257c43ab6419b05dfb2211d0e44284c6",
            "jsonPath": "$.emailAddress",
            "schema": "iglu:com.acme/email_sent/jsonschema/1-0-0"
          }
        ]
      },
      "strategy": {
        "pseudonymize": {
          "hashFunction": "SHA-256"
        }
      }
    }
  }
}
```
In this example there are a few things going on. The PII enrichment was configured to pseudonymize the canonical fields: `user_fingerprint`, `user_ipaddress` and `user_id` and the event contains their original and modified values. In addition, it was configured to pseudonymize fields from the `unstruct_event` and `contexts` fields which are JSON formatted strings. As in the canonical case the event contains the original and modified values, but in addition it contains the schema iglu URL and the JSON path corresponding to it as in the case of `contexts` there could be any number of substitutions depending on the path and schema matches. Finally the strategy and in this case the hashing algorithm version is also given. What is not emitted is the `salt` that was used in the hashing.


[schema]: http://iglucentral.com/schemas/com.snowplowanalytics.snowplow.enrichments/pii_enrichment_config/jsonschema/2-0-0
[gdpr]: https://www.eugdpr.org/
[release-0.9.11]: https://snowplowanalytics.com/blog/2014/11/10/snowplow-0.9.11-released-with-webhook-support/#mailchimp
[pii-config-schema]: https://github.com/snowplow/iglu-central/blob/master/schemas/com.snowplowanalytics.snowplow.enrichments/pii_enrichment_config/jsonschema/2-0-0

[md2]: https://en.wikipedia.org/wiki/MD2_(cryptography)#MD2_hashes
[md5]: https://en.wikipedia.org/wiki/MD5#MD5_hashes
[sha-1]: https://en.wikipedia.org/wiki/SHA-1#Example_hashes
[sha-2]: https://en.wikipedia.org/wiki/SHA-2#Comparison_of_SHA_functions
[rfc6149]: https://tools.ietf.org/html/rfc6149
[stream-enrich-configuration]: https://github.com/snowplow/snowplow/wiki/Configure-Stream-Enrich
[canonical-event-model]: https://github.com/snowplow/snowplow/wiki/canonical-event-model
[analytics-sdk]: https://github.com/snowplow/snowplow/wiki/Snowplow-Analytics-SDK
[stream-enrich-pii-emit]: https://github.com/snowplow/snowplow/tree/master/3-enrich/stream-enrich/core/src/main/scala/com.snowplowanalytics.snowplow.enrich.stream/sources/Source.scala
[parent-event-schema]: https://github.com/snowplow/iglu-central/tree/master/schemas/com.snowplowanalytics.snowplow/parent_event/jsonschema/1-0-0
[pii-transformation-schema]: https://github.com/snowplow/iglu-central/blob/master/schemas/com.snowplowanalytics.snowplow/pii_transformation/jsonschema/1-0-0
