---
layout: post
title: "Flink and Prometheus: Cloud-native monitoring of streaming applications"
date: 2019-02-27T13:00:00.000Z
authors:
- fabian:
  name: "Maximilian Bode"
  twitter: "mxpbode"

excerpt: This blog post describes how developers can leverage Apache Flink’s built-in metrics system together with Prometheus to observe and monitor streaming applications in an efficient way.
---

This blog post describes how developers can leverage Apache Flink’s built-in metrics system together with [Prometheus](https://prometheus.io/) to observe and monitor streaming applications in an efficient way. This is a follow-up post from my [Flink Forward](https://flink-forward.org/) Berlin 2018 talk ([slides](https://www.slideshare.net/MaximilianBode1/monitoring-flink-with-prometheus), [video](https://www.ververica.com/flink-forward-berlin/resources/monitoring-flink-with-prometheus)). We will cover some basic Prometheus concepts and why it is a great fit for monitoring Apache Flink stream processing jobs. There is also an example to showcase how you can utilize Prometheus with Flink to gain insights into your applications and be alerted on potential degradations of your Flink job.<br>
<br>

## WHY PROMETHEUS 
<br>

Prometheus is an open-source metrics-based monitoring system that was originally created in 2012. The system is completely open-source with a vibrant community behind it and it has graduated from the Cloud Native Foundation last year – a sign of maturity, stability and production-readiness. As we mentioned, the system is based on metrics and it is designed to measure the overall health, behavior, and performance of a service. Prometheus features a multi-dimensional data model as well as a flexible [query language](https://prometheus.io/docs/prometheus/latest/querying/basics/). It is highly performant and can easily be deployed in traditional or containerized environments. Some of the important Prometheus concepts are:<br>
<br>

* **Metrics:** Prometheus defines metrics as floats of information that change in time. These time series have millisecond precision.<br>
<br>

* **Labels** are the key-value pairs associated with time series that support Prometheus’ flexible and powerful data model – in contrast to hierarchical data structures that one might experience with traditional metrics systems.<br>
<br>

* **Scrape:** Prometheus is a pull-based system and fetches (“scrapes”) metrics data from specified sources that expose HTTP endpoints with a text-based format.<br>
<br>

* **PromQL** is Prometheus’ query language that can be used for both building dashboards and setting up alert rules that will trigger when specific conditions are met.<br>
<br>

When considering metrics and monitoring systems for your Flink jobs, there is a [wide variety of options that can be used](https://ci.apache.org/projects/flink/flink-docs-stable/monitoring/metrics.html). Flink offers native support for exposing data to Prometheus via the PrometheusReporter configuration. Setting up this integration is very easy.

Prometheus is a great choice as usually Flink jobs are not running in isolation but in a greater context e.g. of microservices. For making metrics available to Prometheus from other parts of a larger system, there are two options: There exist [libraries for all major languages](https://prometheus.io/docs/instrumenting/clientlibs/) to instrument other applications and there is a wide variety of [exporters](https://prometheus.io/docs/instrumenting/exporters/), i.e. tools that expose metrics of third-party systems (like databases or Apache Kafka) as Prometheus metrics.<br>
<br>

## PROMETHEUS AND FLINK IN ACTION 
<br>

We have provided a [GitHub repository](https://github.com/mbode/flink-prometheus-example) that showcases the integration described above. To have a look, make sure [Docker](https://docs.docker.com/install/) is available, clone the repository and run: <br>
<br>
```java
./gradlew composeUp
```
This builds a Flink job using [Gradle](https://gradle.org/) and starts up a local environment based on [Docker Compose](https://docs.docker.com/compose/) running the job in a [Flink job cluster](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/deployment/docker.html#flink-job-cluster) (reachable at [http://localhost:8081](http://localhost:8081/)) as well as a Prometheus instance ([http://localhost:9090](http://localhost:9090/)).<br>
<br>

<br>
<center>
<img src="{{ site.baseurl }}/img/blog/prometheusexamplejob.png" width="600px" alt="flink-prometheus-example"/>
</center>
<br>

A simple demo illustrates how easy it is to add custom metrics relevant to your business logic into your Flink job. The <code>PrometheusExampleJob</code> has three operators: Random numbers up to 10,000 are generated, then a map counts the events and creates a histogram of the values passed through. Finally, the events are discarded without further output.

## CONFIGURING PROMETHEUS WITH FLINK

To start monitoring Flink with Prometheus, the following steps are necessary:
<br>

1 Make the <code>PrometheusReporter</code> jar available to the _classpath_ of the Flink cluster:

<br>

```java
cp /opt/flink/opt/flink-metrics-prometheus-1.7.0.jar /opt/flink/lib
```
<br>

2 [Configure the reporter](https://ci.apache.org/projects/flink/flink-docs-stable/monitoring/metrics.html) in Flink’s _flink-conf.yaml_. All job managers and task managers will expose the metrics on the configured port.

<br>

```java
metrics.reporters: prom

metrics.reporter.prom.class: 

org.apache.flink.metrics.prometheus.PrometheusReporter

metrics.reporter.prom.port: 9999
```
<br>

3 Prometheus needs to know where to scrape metrics. In a static scenario, you can simply [configure Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) in _prometheus.yml_ with the following:

<br>

```java
scrape_configs
  - job name: ‘flink’
    static configs:
     - targets: [‘job-cluster:9999’, ‘taskmanager1:9999’, ‘taskmanager2:9999’]
```
<br>

In more dynamic scenarios we recommend using Prometheus’ service discovery support for different platforms such as Kubernetes, AWS EC2 and more.

Both custom metrics are now available in Prometheus:

<br>
<center>
<img src="{{ site.baseurl }}/img/blog/prometheus.png" width="600px" alt="flink-prometheus-example"/>
</center>
<br>

More technical metrics from the Flink cluster (e.g. checkpoint sizes or duration, Kafka offsets or resource consumption) are also available. To test Prometheus’ alerting feature, kill one of the Flink task managers via

```java
docker kill taskmanager1
```

After roughly one minute (as configured in the alert rule), the following alert will fire:

<br>
<center>
<img src="{{ site.baseurl }}/img/blog/prometheusalerts.png" width="600px" alt="flink-prometheus-example"/>
</center>
<br>

In real-world situations these alerts can be routed through a component called [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) and be grouped into notifications to e.g. email, PagerDuty or Slack.

Feel free to play around with the setup, and check out the [Grafana](https://grafana.com/grafana) instance reachable at [http://localhost:3000](http://localhost:3000/) (credentials _admin:flink_) for visualizing Prometheus metrics. If there are any questions or problems, do not hesitate to [create an issue](https://github.com/mbode/flink-prometheus-example/issues). Once finished, do not forget to tear down the setup via

```java
./gradlew composeDown
```

## CONCLUSION

Using Prometheus together with Flink provides an easy way for effective monitoring and alerting for your Flink jobs. Both projects have exciting and vibrant communities behind them with new developments and additions scheduled for upcoming releases. We encourage you to try the two technologies together as it has immensely improved our visibility into Flink jobs running in production.
