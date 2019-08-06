<a name="top" />

[**HOME**](Home) » [**SNOWPLOW TECHNICAL DOCUMENTATION**](Snowplow-technical-documentation) » [**Snowplow Analytics SDK**](Snowplow-Analytics-SDK) » Scala Analytics SDK setup

## Contents

- 1 [Overview](#overview)  
- 2 [Compatibility](#compatibility)  
- 3 [Setup](#setup)  
  - 3.1 [SBT](#sbt)  
  - 3.2 [Gradle](#gradle)  
  - 3.3 [Maven](#maven)  

<a name="overview" />

### 1. Overview

The [Snowplow Analytics SDK for Scala][repo] lets you work with [Snowplow enriched events](canonical-event-model) in your Scala (or JVM) event processing, data modeling and machine-learning jobs. 
You can use this SDK with [Apache Spark](http://spark.apache.org/), [AWS Lambda](https://aws.amazon.com/lambda/), [Apache Flink](https://flink.apache.org/), 
[Scalding](https://github.com/twitter/scalding), [Apache Samza](http://samza.apache.org/) and other JVM-compatible data processing frameworks.

<a name="compatibility" />

### 2. Compatibility

Snowplow Scala Analytics SDK was compiled against Scala versions 2.10, 2.11 and 2.12.
Minimum required Java Runtime is JRE7.

Current version of Scala Analytics SDK includes Json4s version 3.2.11, which is binary [incompatible][json4s-binary-compat] with Json4s versions included in Spark 1.x, though it is fully compatible with Spark 2.x and beyond.
With Spark 1.x you can use version <= 0.2.0.

<a name="setup" />

### 3. Setup

The latest version of Snowplow Scala Analytics SDK is 0.4.2 and it is available on Maven Central.

<a name="sbt" />

### 3.1 SBT

If you’re using SBT, add the following lines to your build file:

```scala
// Dependency
libraryDependencies += "com.snowplowanalytics" %% "snowplow-scala-analytics-sdk" % "0.4.2"
```

Note the double percent (`%%`) between the group and artifactId. This will ensure that you get the right package for your Scala version.

<a name="gradle" />

### 3.2 Gradle

If you are using Gradle in your own job, then add following lines in your `build.gradle` file:

```scala
dependencies {
    ...
    // Snowplow Scala Analytics SDK
    compile 'com.snowplowanalytics:snowplow-scala-analytics-sdk_2.11:0.4.2'
    }
```

Note that you need to change `_2.11` to `_2.10` or `_2.12` in artifactId if you're using other Scala versions.

<a name="maven" />

### 3.3 Maven

If you are using Maven in your own job, then add following lines in your `pom.xml` file:

```xml
<dependency>
    <groupId>com.snowplowanalytics</groupId>
    <artifactId>snowplow-scala-analytics-sdk_2.11</artifactId>
    <version>0.4.2</version>
</dependency>
```

Note that you need to change `_2.11` to `_2.10` or `_2.12` in artifactId if you're using other Scala versions.

Done? Now read the [Scala Analytics SDK API](Scala-Analytics-SDK) to start analyzing events data.

[repo]: https://github.com/snowplow/snowplow-scala-analytics-sdk
[json4s-binary-compat]: https://github.com/json4s/json4s/issues/212
