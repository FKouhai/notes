---
id: monitoring-patterns
aliases:
  - monitoring-patterns
tags:
  - books
  - sre
  - monitoring
---

# monitoring-patterns

### Composable Monitoring

Usage of multiple specialized tools and couple them loosely together, creating a monitoring platform, some tools are:

- Graphite
- Sensu
- Logstash
- Collectd

#### Components of a monitoring system

- Data colelction -> Component that collects the data, composed of metrics which could be of type counter,gauge or logs
- Data storage -> Component that saves that data, could be a tsdb
- Visualization -> Component that allows the operator/engineer to view the data
- Analytics and reporting -> Usage of SLA
- Alerting -> Component that triggers an alert to a certain communication endpoint based on a set of metrics

### Monitor from the user perspective

What this means is, monitor the possible errors a user could have, for example how many HTTP 5XX codes does the endpoint returns 

### Buy, not build

Using a SaaS monitoring solution is cheaper, taking into account that if your team/company builds a monitoring platform it will have a big engineering expense,SaaS allows you to 
focus on the product, and you are most likely not an expert at architecting these kind of tools, there are only 2 real reasons not to use SaaS

1) You've outgrown it
2) There are some security and compliance reasons that forbids you to use a SaaS platform

### Continual Improvement

Great monitoring doesnt come in a week effort, it is something prolongued in time where your team learnt from its failures
