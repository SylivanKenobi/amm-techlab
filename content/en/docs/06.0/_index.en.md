---
title: "6. Observability"
weight: 6
sectionnumber: 6
description: >
  Monitoring and tracing of applications running on OpenShift.
---


{{% alert title="Warning" color="secondary" %}}Dieses Lab muss noch überarbeitet werden{{% /alert %}}


## TODO Lab Monitroing

* [ ] Java App deployen [die hier nehmen](https://gitlab.puzzle.ch/craaflaub/techlab/-/blob/master/07-prometheus.md) oder auf [unsere eigene Springboot app](https://github.com/appuio/example-spring-boot-helloworld) erweitern
* [ ] Service Monitor erstellen. Integration in Plattform Prometheus (mit baf schauen)
* [ ] Prometheus Web Console durchgehen, Targets, usw. anschauen, ein paar Queries ausführen
* [ ] Grafana und ein Dashboard erstellen auf basis der Daten in der App


## TODO Lab Tracing

* [ ] [Tracing Demo](https://github.com/RedHat-Middleware-Workshops/quarkus-workshop/blob/master/docs/tracing.adoc) übernehmen.


### Notes Jaeger

Deploy Jaeger:

```s

oc apply -f jaeger.yaml

```

Deploy Kafka:

```s

oc apply -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/content/en/docs/03.0/3.2/kafka-cluster.yaml
oc apply -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/content/en/docs/03.0/3.2/manual-topic.yaml

```

Deploy Microservices:

```s

oc apply -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/content/en/docs/06.0/data-consumer.yaml
oc apply -f https://raw.githubusercontent.com/puzzle/amm-techlab/master/content/en/docs/06.0/data-producer.yaml

```
