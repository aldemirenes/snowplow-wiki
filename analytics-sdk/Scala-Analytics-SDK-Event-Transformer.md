<a name="top" />

[**HOME**](Home) » [**SNOWPLOW TECHNICAL DOCUMENTATION**](Snowplow-technical-documentation) » [**Snowplow Analytics SDK**](Snowplow-Analytics-SDK) » Scala Analytics SDK

*This page refers to version 0.4.0 of the Snowplow Scala Analytics SDK.*  
*Click [here](https://github.com/snowplow/snowplow/wiki/Scala-Analytics-SDK-Event-Transformer-0.3.0) for the corresponding documentation for version 0.3.0.*  

- 1 [Overview](#overview)  
- 2 [The JSON Event Transformer](#transformer)  
- 3 [Examples](#example)  
  - 3.1 [Using from Apache Spark](#spark)  
  - 3.2 [Using from AWS Lambda](#lambda)  

<a name="overview" />

## Overview

The Snowplow enriched event is a relatively complex TSV string containing scalars and self-describing JSONs.
Rather than work with this structure directly, Snowplow analytics SDKs ship with *event transformers*, which translate the Snowplow enriched event format into other formats that are more convenient for engineers and analysts.

As the Snowplow enriched event format evolves towards a cleaner [Apache Avro](https://avro.apache.org/)-based structure, we will be updating this SDK to maintain compatibility across different enriched event versions.

Working with the Snowplow Scala Analytics SDK therefore has two major advantages over working with Snowplow enriched events directly:

1. The SDK reduces your development time by providing analyst- and developer-friendly transformations of the Snowplow enriched event format;
2. The SDK futureproofs your code against new releases of Snowplow which update our enriched event format.

Currently the Analytics SDK for Scala ships with one event transformer: the JSON Event Transformer. 

<a name="transformer" />

## The JSON Event Transformer

The JSON Event Transformer takes a Snowplow enriched event and converts it into a JSON ready for further processing. This transformer was adapted from the code used to load Snowplow events into Elasticsearch in the Kinesis real-time pipeline.

The JSON Event Transformer converts a Snowplow enriched event into an instance of the `Event` case class, a representation of a canonical Snowplow event, like so:

```scala
Event(
  app_id = Some("angry-birds"),
  platform = Some("web"),
  etl_tstamp = Some(Instant.parse("2017-01-26T00:01:25.292Z")),
  collector_tstamp = Instant.parse("2013-11-26T00:02:05Z"),
  dvce_created_tstamp = Some(Instant.parse("2013-11-26T00:03:57.885Z")),
  event = Some("page_view"),
  event_id = UUID.fromString("c6ef3124-b53a-4b13-a233-0088f79dcbcb"),
  txn_id = Some(41828),
  name_tracker = Some("cloudfront-1"),
  v_tracker = Some("js-2.1.0"),
  v_collector = "clj-tomcat-0.1.0",
  v_etl = "serde-0.5.2"
  /* ... */
)
```

This case class can be rendered into a JSON object, and subsequently a JSON string, or worked with to interact with the event's fields in a typesafe manner.

The most complex piece of processing is the handling of the self-describing JSONs found in the enriched event's `unstruct_event`, `contexts` and `derived_contexts` fields. Currently  there are two alternative behaviors for handling them in the event transformer: 

1. Under the original "lossy" behavior, if an enriched event contained a `com.snowplowanalytics.snowplow/link_click/jsonschema/1-0-1`, then the unstructured event field would be rendered in the final JSON like this:

```json
"unstruct_event_com_snowplowanalytics_snowplow_link_click_1": {
  "targetUrl": "http://www.example.com",
  "elementClasses": ["foreground"],
  "elementId": "exampleLink"
}
```

2. Under the new "lossless" behavior, available since 0.3.1, if an enriched event contained a `com.snowplowanalytics.snowplow/link_click/jsonschema/1-0-1`, then the final JSON (if turned into a string) would contain a self-describing JSON object instead:

```json
"unstruct_event": {
  "schema": "iglu:com.snowplowanalytics.snowplow/link_click/1-0-1",
  "data": {
    "targetUrl": "http://www.example.com",
    "elementClasses": ["foreground"],
    "elementId": "exampleLink"
  }
}
```

Along with the `Event` case class, the JSON Event Transformer comes with the following functions:

- `Event.parse(line)` - similar to the old `transform` function, this method accepts an enriched Snowplow event as a string and returns an `Event` instance as a result.
- `event.toJson(lossy)` - similar to the old `getValidatedJsonEvent` function, it transforms an `Event` into a validated JSON whose keys are the field names corresponding to the EnrichedEvent POJO of the Scala Common Enrich project. If the lossy argument is true, any self-describing events in the fields (unstruct_event, contexts and derived_contexts) are returned in a "shredded" format, e.g. `"unstruct_event_com_acme_1_myField": "value"`. If it is set to false, they are not flattened into underscore-separated top-level fields, using a standard self-describing format instead.
- `event.inventory` - extracts metadata from the event containing information about the types and Iglu URIs of its shred properties (unstruct_event, contexts and derived_contexts). Unlike version 0.3.0, it no longer requires a `transformWithInventory` call and can be obtained from any `Event` instance.
- `atomic` - returns the event as a map of keys to Circe JSON values, while dropping inventory fields. This method can be used to modify an event's JSON AST before converting it into a final result.
- `ordered` - returns the event as a list of key/Circe JSON value pairs. Unlike `atomic`, which has randomized key ordering, this method returns the keys in the order of the canonical event model, and is particularly useful for working with relational databases.
- `circeJson.as[Event]` - Encoded [Circe](circe) Event JSON can be decoded to Event since Event decoder is provided in the SDK but bear in mind it works only for non-lossy encoding.

### Event inventory

An event's inventory is simply a list of metadata about its shredded types:

1. Where it was extracted from: `unstruct_event` column (`UnstructEvent`), `contexts` column (`Contexts(CustomContexts)`) or `derived_contexts` column (`Contexts(DerivedContexts)`)
2. Its Iglu URI (e.g. `iglu:com.acme/context/jsonschema/1-0-0`), stored as an Iglu `SchemaKey` instance.

<a name="example" />

## Examples

<a name="spark" />

### Using from Apache Spark

The Scala Analytics SDK is a great fit for performing Snowplow [event data modeling](http://snowplowanalytics.com/blog/2016/03/16/introduction-to-event-data-modeling/) in Apache Spark and Spark Streaming.

Here’s the code we use internally for our own data modeling jobs:

```scala
import com.snowplowanalytics.snowplow.analytics.scalasdk.Event

val events = input
  .map(line => Event.parse(line))
  .filter(_.isValid)
  .flatMap(_.toOption)
  .map(event => event.toJson(true).noSpaces)

val dataframe = spark.read.json(events: _*)
```

<a name="lambda" />

### Using from AWS Lambda

The Scala Analytics SDK is a great fit for performing analytics-on-write, monitoring or alerting on Snowplow event streams using AWS Lambda.

Here’s some sample code for transforming enriched events into JSON inside a Scala Lambda:

```scala
import com.snowplowanalytics.snowplow.analytics.scalasdk.Event

def recordHandler(event: KinesisEvent) {

  val events = for {
    rec <- event.getRecords
    line = new String(rec.getKinesis.getData.array())
    event = Event.parse(line)
  } yield event

  /* ... */
}
```

[Back to top](#top)  
[Back to Scala Analytics SDK contents][contents]

[contents]: Scala-Analytics-SDK
[circe]: https://circe.github.io/circe/
