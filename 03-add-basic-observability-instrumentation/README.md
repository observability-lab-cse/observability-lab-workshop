# Adding Basic Observability Instrumentation

> Goal: Exploring different options to instrument applications with standard tooling and easily integrate them into Azure for data collection and observability.

<!--
In this section we are going to build the observability solution step by step by using automatic OpenTelemetry instrumentation.

1. OtelCollector introduction and ref to lower section with comparison to alertnatives
2. Auto instrumentation of .Net application
3. Auto instrumentation of Java application

Additional topics:

- OtelCollector and the alternatives quick comparison. -->

## Opentelemetry Collector

There are many ways applications can be instruments. Some require more custom code some less.

> Note: The [section below](TODO) contains a comparison of different approachs on how to instrument applications as well as links to relevant resources or samples.

For this workshop lest use a zero code approach as it will come in handy when working with preexisting application, which is the [Otel Collector](TODO).

## Configure Applications

For our appication to send out telemetry data to the collectore, we need to run a small "agent' withing the conatienr that will gatter the telemetry from standart libarbies.

Lets have a look at this for the Java application. Here is a link to the [Otel Autosinstrumentation](https://github.com/open-telemetry/opentelemetry-java-instrumentation). Try to adjust the applications docker file and configs in such a way that we can scrape the logs and send them to the otel collector.
