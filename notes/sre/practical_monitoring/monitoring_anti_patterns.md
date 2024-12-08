---
id: monitoring_principles
aliases:
  - monitoring_principles
tags:
  - books
  - sre
  - anti-patterns
---


# Monitoring Anti-Patterns

### Tool Obsession

Being in a tool driven team wont be as effective as a mission driven team, there is no such thing a silver bullet, no tool is perfect
Dont stick to a single tool if it cant suit most of your needs or all of those, this prevents vendor lock in as well, it is needed to take into account
everything that needs to be monitored, from physical hardware, network to the application(s) (it/them)self

Monitoring is not a single problem, when it comes down to monitoring it would be better to think about it in terms of platform, you have a monitoring platform
with all the tools you need, for example prometheus, loki, thousand eyes, etc ...


#### The observer effect isnt a problem

Nowadays worrying about the load a monitoring system may have on the rest of the system should not be a thing, since we have bigger hardware than we used to have

#### How many tools is too many?

There isn't a rule of thumb where we could state how many are too many, just use the less possible to get the job done and try to have them as coupled as possbile, for example 
a dashboard and log viewer like grafana where you can get many datasources for the metrics and logs is a really useful tool

#### Avoid cargo culting tools

Dont adopt a tool just because a team more succesful than yours uses it without making any proper testing on the tool it is always important to know what tool are you adopting,
and if adopting it will add new things to the table


#### Sometimes you really have to build it

There may be times where a tool wont satisfy your needs, and there arent any tools that could do what you need so you may have to build it from scratch


#### The single pane of glass is a myth

You may need to have multiple tools feeding into multiple dashboards, or having different dashboards that are fed by one tool


### Monitoring as a job

Monitoring is not a job, you cannot monitor what you dont understand, sometimes being too specialized on something is not the best option if its a role, for example monitoring engineer, monitoring is a skill, just like automation or networking, everyone within your team should understand monitoring and have it within their skillset




### Checkbox monitoring

Checkbox monitoring is when you have monitoring systems for the sole sake of saying have them, maybe some higher up made it a requirement, this ends up with an ineffective, noisy, untrustworthy and probably worse than having no monitoring

Some common signs may be:

- Recording metrics like system load,CPU usage and memory usage but the service still goes down without you knowing why
- You end up consistently ignoring alerts as they are false alarms more often than not
- You are checking systems for metrics every 5 mins or less
- You are not storing historical metric data


#### What does working actually mean

This comes down to understand what you are monitoring and understand the context of the application itself


#### OS Metrics aren't very useful for alerting

Some services we run are resource intensive by nature and thats OK, it is way more beneficial to alert on what working means as opposed to low level metrics

#### Collect your metrics more often

You need to collect metrics at least every 60 seconds, if they are high traffic applications, then opt for more often, take into account the retention policy you may need for the recollection, does it really make any sense to collect 1 year metrics over 10 second periods


### Using monitoring as a Crutch

Monitoring is important but it doesnt solve your problems, after having alerting and monitoring in place it is important to know what is needed to be fixed

### Manual Configuration

Monitoring tools, should be fully automated, a service or a system should be able to register itself to the monitoring tool, instead of doing it manually


#### Runbook abuse

If the runbook used to heal a system is just a set of manual steps the monitoring system should be able to run that runbook before alerting you


































































