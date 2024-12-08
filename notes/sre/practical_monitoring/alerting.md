---
id: alerting
aliases:
  - alerting
tags:
  - books
  - sre
  - monitoring
---

# Alerts, On-Call and Incident Management

Alerts are extremely important to know when something goes wrong, without them we would be spending most of our time looking at graphs and dashboards

Alerting is not a trivial thing, you could have metrics that thend to spike, and having an alert rule over that could trigger false positives, aggregating
all the alerts into a single point within 5 minutes, makes us lose granularity

## What makes a good alert?

They are 2 different kind of alerts

1) Alerts meant to wake someone up
2) Alerts as an FYI

The best practices when it comes down to alerts are:

1) [Stop using email for alerts](#stop-using-email-for-alerts)
2) [Write runbooks](#write-runbooks)
3) [Arbitrary static thresholds arent the only way](#arbitrary-static-thresholds-arent-the-only-way) 
4) [Delete and tune alerts](#delete-and-tune-alerts) 
5) [Use maintenance periods](#use-maintenance-periods) 
6) [Attempt self-healing first](#attempt-automated-self-healing-first) 

#### Stop using email for alerts

An email is not going to wake someone up, so think about the usecases for your alerts

1) Reponse/action required immediately -> Actual alert, send this to your pager either SMS,PagerDuty etc
2) Awareness needed. but immediate action not required -> Send this to chat rooms 
3) Record for historical/diagnostic purposes -> Send the information to a log file

##### Logging your alerts

It is important to retain logs of your alerts, so you can report on them later

### Write runbooks

A runbook is a way to quickly orient yourself when an alert fires a good runbook is written for a particular service and answers several questions

- What is this service, and what does it do?
- Who is responsible for it?
- What dependencies does it have?
- What does the infrastructure for it look like?
- What metrics and logs does it emit and what do they mean?
- What alerts are set up for it?

For every alert it is important to include a link to the runbook for that service

### Arbitrary static thresholds arent the only way

Not every situation has a warning and critical state that makes sense, there are cases where data point X > Y is not useful at all for example having a threshold set at free space under 10% which would make you miss a disk quickly growing in usage from 10% to 80%


### Delete and Tune Alerts

Noisy alerts cause people to stop trusting the monitoring system, which leads people to completely ignoring the alerts, in order to reduce the alerts do the following

1) Do all the alerts require someone to act?
2) Look at a month's worth of history for your alerts, what are they?, are there alerts that could be removed?
3) What automation can you build to make the alert obsolete entirely?


### Use Maintenance periods

If you have an alert that is monitoring something you are working on which could lead to a timeout, have that alert silenced, it doesnt make much sense to send the alert over

### Attempt Automated Self-Healing First

If the most common action needed on an alert is to perform a known and documented series of steps which usually fixes the problem, automate that, and let the alerting system to run the script

## On-Call

Being on call means that you have to be available to respond to issues after hours, to reduce the on-call time:

### Fixing False Alarms

Although its impossible to have a 100$ accurate alerting system, some can be cut down by doing the following:

#### Cutting down on needless firefighting

There are 2 effective strategies to get into this habit:

1) Make it the duty of on call to work on systems resiliency and stability during their on call shift when they arent fighting fires
2) Explicitly plan for systems resiliency and stability work during the following week's sprint planning


#### Building a better on call rotation

It is better to start an on call rotation on a week day isntead of a calendar week to do an on call handoff


## Incident Management

Use ITIL

1) Incident identification
2) Incident logging
3) Incident diagnosis, categorization,resolution and closure
4) Communications throughout the event
5) After the incident is resolved come up with the remediation plans for building in more resiliency


## Postmortems

A post mortem is a report after a service failure/downtime, this report has to be detailed with a timeline of events, who was the person in charge, lessons learnt and how the issue was fixed


