<a name="top" />

[**HOME**](Home) > [**SNOWPLOW SETUP GUIDE**](Setting-up-Snowplow) > [**Step 4: setting up alternative data stores**](Setting-up-alternative-data-stores) > [Snowplow-S3-Loader](Snowplow-S3-Loader-Setup)

This documentation is for version 0.6.0 of Snowplow S3 Loader. For previous versions:

* **[Version 0.5.0][v0.5]**
* **[Version 0.4.0][v0.4]**
* **[Version 0.3.0][v0.3]**
* **[Version 0.1.0 - 0.2.1][v0.1]**

## Overview

Snowplow S3 Loader reads records from an [Amazon Kinesis][kinesis] or [NSQ][nsq] stream, compresses them
using [splittable LZO][splittable-lzo] or [GZip][gzip], and writes them to S3.

It was created to store the [Thrift][thrift] records generated by the [Scala Stream Collector][ssc]
in S3.

If it fails to process a record, it will write that record to a second Kinesis or NSQ stream along with an
error message.

## Prerequisites

To run the Snowplow S3 Loader, you must have installed the native LZO binaries. To do this on Ubuntu,
run:

```
$ sudo apt-get install lzop liblzo2-dev
```

## Installation

### Option 1: download the executable jar file

See [[Hosted assets]] for the zipfile to download.

### Option 2: compile from source

Alternatively, you can build it from the source files. To do so, you will need [scala][scala] and
[sbt][sbt] installed.

To do so, clone the Snowplow repo:

```
$ git clone https://github.com/snowplow/snowplow-s3-loader.git
```

Navigate into the snowplow-s3-loader folder:

```
$ cd snowplow-s3-loader
```

Use `sbt` to resolve dependencies, compile the source, and build an assembled fat JAR file with all
dependencies.

```
$ sbt assembly
```

The `jar` file will be saved as `snowplow-s3-loader-0.6.0.jar` in the `target/scala-2.11` subdirectory.
It is now ready to be deployed.

## Configuration

The sink is configured using a HOCON file. These are the fields:

* `source`: Choose kinesis or nsq as a source stream
* `sink`: Choose between kinesis or nsq as a sink stream for failed events
* `aws.accessKey` and `aws.secretKey`: Change these to your AWS credentials. You can alternatively leave them as "default", in which case the [DefaultAWSCredentialsProviderChain][DefaultAWSCredentialsProviderChain] will be used.
* `kinesis.initialPosition`: Where to start reading from the stream the first time the app is run. "TRIM_HORIZON" for as far back as possible, "LATEST" for as recent as possibly, "AT_TIMESTAMP" for after the specified timestamp.
* `kinesis.initialTimestamp`: Timestamp for "AT_TIMESTAMP" initial position
* `kinesis.maxRecords`: Maximum number of records to read per GetRecords call
* `kinesis.region`: The Kinesis region name to use.
* `kinesis.appName`: Unique identifier for the app which ensures that if it is stopped and restarted, it will restart at the correct location.
* `nsq.channelName`: Channel name for NSQ source stream. If more than one application reading from the same NSQ topic at the same time, all of them must have unique channel name to be able to get all the data from the same topic.
* `nsq.host`: Hostname for NSQ tools
* `nsq.port`: HTTP port number for nsqd
* `nsq.lookupPort`: HTTP port number for nsqlookupd
* `stream.inStreamName`: The name of the input stream of the tool which you choose as a source. This should be the stream to which your are writing records with the Scala Stream Collector.
* `streams.outStreamName`: The name of the output stream of the tool which you choose as sink. This is stream where records are sent if the compression process fails.
* `streams.buffer.byteLimit`: Whenever the total size of the buffered records exceeds this number, they will all be sent to S3.
* `streams.buffer.recordLimit`: Whenever the total number of buffered records exceeds this number, they will all be sent to S3.
* `streams.buffer.timeLimit`: If this length of time passes without the buffer being flushed, the buffer will be flushed.
__Note__: With NSQ streams, only record limit is taken into account. Other two option will be ignored.
* `s3.region`: The AWS region for the S3 bucket
* `s3.bucket`: The name of the S3 bucket in which files are to be stored
* `s3.format`: The format the app should write to S3 in (`lzo` or `gzip`)
* `s3.maxTimeout`: The maximum amount of time the app attempts to PUT to S3 before it will kill itself

### Monitoring

You can also now include Snowplow Monitoring in the application.  This is setup through a new section at the bottom of the config.  You will need to ammend:

+ `monitoring.snowplow.collectorUri` insert your snowplow collector URI here.
+ `monitoring.snowplow.appId` the app-id used in decorating the events sent.

If you do not wish to include Snowplow Monitoring, remove the entire `monitoring` section from the config.

An [example][conf-example] is available in the repo.

## Execution

The Snowplow S3 Loader is a jarfile. Simply provide the configuration file as a parameter:

```
$ java -jar snowplow-s3-loader-0.6.0.jar --config my.conf
```

This will start the process of reading events from selected source, compressing them, and writing them to S3.

## Further reading

- [Kinesis at-least-once processing](Kinesis-at-least-once-processing)

[ssc]: https://github.com/snowplow/snowplow/tree/master/2-collectors/scala-stream-collector
[thrift]: https://thrift.apache.org/
[kinesis]: http://aws.amazon.com/kinesis/
[nsq]: http://nsq.io
[s3]: http://aws.amazon.com/s3/
[splittable-lzo]: http://blog.cloudera.com/blog/2009/11/hadoop-at-twitter-part-1-splittable-lzo-compression/
[gzip]: http://www.gzip.org/
[DefaultAWSCredentialsProviderChain]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/DefaultAWSCredentialsProviderChain.html
[conf-example]: https://github.com/snowplow/snowplow-s3-loader/blob/master/examples/config.hocon.sample
[scala]: http://www.scala-lang.org/
[sbt]: http://www.scala-sbt.org/

[v0.1]: https://github.com/snowplow/snowplow/wiki/kinesis-lzo-s3-sink-setup-0.1.0
[v0.3]: https://github.com/snowplow/snowplow/wiki/kinesis-lzo-s3-sink-setup-0.3.0
[v0.4]: https://github.com/snowplow/snowplow/wiki/kinesis-lzo-s3-sink-setup-0.4.0
[v0.5]: https://github.com/snowplow/snowplow/wiki/kinesis-lzo-s3-sink-setup-0.5.0