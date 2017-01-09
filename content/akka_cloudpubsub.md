Title: Cloud Pub/Sub With Akka Streams
Date: 2016-11-30 18:05
Category: Programming
Tags: Scala, Akka, Google Cloud


I finally managed to open-source the Cloud Pub/Sub Akka Streams components I wrote for the "Manifold" project at work (more details on that project will be published soon). These components have been in production for several months and is used to process a Cloud Pub/Sub firehose that streams over a billion events a day.

Artifacts will be published to Maven Central soon. In the mean time, check out the sources from [https://github.com/QubitProducts/akka-cloudpubsub](https://github.com/QubitProducts/akka-cloudpubsub) and build locally with `sbt publishLocal`.

