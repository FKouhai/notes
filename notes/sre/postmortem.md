---
id: postmortem
aliases: []
tags:
  - expatx
  - sre
---

> Template from: Betsy Beyer, Chris Jones, Jennifer Petoff, and Niall Richard Murphy. [“Site Reliability Engineering.”](https://landing.google.com/sre/book/chapters/postmortem.html).

# EKS FAILING DEPLOYMENTS

### Date
06-07-2024

### Authors
- Fran Cirka

### Status
- Fixed

### Summary
An issue has been detected where the redeployments on the staging env weren't working affecting the dev team in order to progress with the testing

### Impact
As of right now staging is the environment live, therefore the impact is HIGH, the dev team was unable to test the new features and improvements within the microservices for expatx

### Root Causes
- Outdated kubernetes version
- Outdated CNI add-on version

### Trigger
- Manual trigger when redeploying from a CI/CD

### Resolution
The issue was fixed after updating the kubernetes and CNI add-on versions

### Detection
Issue was detected by Jake and Fran when checking out the deployment status using k9s

## Action Items
- Updated kubernetes cluster version
- Updated node-group kubernetes version
- Updated CNI add-on version

## Lessons Learned
mainly having the CNI version matching the current kubernetes version in order to prevent bugs like this within the infrastructure, having a monitoring system would've helped detecting the issue earlier and probably having a runbook to automate the fix for this issue

### What went well
The errors were fixed relatively fast and now the services are up and running on staging 

### What went wrong
It took us some time to check the cni errors, the node seemed to be alright so it was a bit of work that could've been much faster

