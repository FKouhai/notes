---
id: 001-Architecture
aliases:
  - argocd-architecture
tags:
  - argocd
  - kubernetes
  - gitops
  - sre
---

# Argo CD Architecture

## Overview

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. It follows the GitOps pattern of using Git as a source of truth for defining the desired application state. Argo CD automates the deployment of applications to Kubernetes clusters by continuously monitoring running applications and comparing their live state against the desired target state defined in Git repositories.

## Core Components

### 1. API Server
The main interface to the Argo CD, responsible for:
- Handling all API calls
- Managing application and repository state
- Providing the UI backend
- Processing Webhook events

### 2. Repository Server
Internal service which maintains a local cache of the Git repository holding the application manifests:
- Clones and caches Git repositories
- Generates and caches Kubernetes manifests using templating tools
- Refreshes repository state at fixed intervals

### 3. Application Controller
Kubernetes controller which continuously monitors running applications and compares current state against desired state:
- Watches for application state changes
- Detects drift between desired and live states (Out of Sync)
- Executes sync operations to reconcile differences
- Reports health status of applications

### 4. Redis
Used as a cache store to store application cache and session information.

## Architecture Diagram
```
                    ┌─────────────┐
                    │   Browser   │
                    └──────┬──────┘
                           │ HTTPS/WebSocket
                    ┌──────┴──────┐
                    │ API Server  │
                    └─┬─────────┬─┘
                      │         │
         git repo     │         │ k8s
      ┌──────────┐    │         │ cluster
      │          ├───▶│         │ watches
      │  Git     │    │         │ apps
      │          │    │         ▼
      └──────────┘    │  ┌──────────────┐
                      │  │ App          │
      ┌──────────┐    │  │ Controller   │
      │          │    │  └──────────────┘
      │ kubectl  ├───▶│         │
      │          │    │         │
      └──────────┘    │         │
                      ▼         ▼
              ┌─────────────────────────┐
              │    Repository Server    │
              └─────────────────────────┘
                        │
                        │ redis
                        ▼
              ┌─────────────────────────┐
              │          Redis          │
              └─────────────────────────┘
```

## Key Concepts

### Application
An application represents a deployed service in Kubernetes represented by a combination of:
- Source Repository URL
- Source Revision (Branch/Tag/Commit)
- Path in the repository containing the manifests
- Destination Cluster URL
- Destination Namespace

### Project
Logical grouping of applications, useful for:
- Restricting sources from which applications can be deployed
- Restricting destinations to which applications may be deployed
- Grouping applications by team, environment, or tenants

### Cluster
A Kubernetes cluster registered with Argo CD that can deploy applications.

### Repository
A Git repository containing the application manifests.

## Deployment Models

### Helm Charts
Supports Helm charts as application sources with the ability to override values.

### Kustomize
Native support for Kustomize with parameter overrides.

### Plain YAML/Jinja2
Support for plain Kubernetes YAML manifests with support for Jinja2 templating.

### Custom Tools
Support for custom configuration management tools using config management plugins.

## Communication Flow

1. User defines desired state in Git repository
2. Argo CD Repository Server clones and caches the repository
3. Application Controller continuously compares desired vs live state
4. Out-of-sync applications detected and reported through metrics/UI
5. Manual or automatic sync reconciles differences
6. Changes propagated to Kubernetes cluster through kubectl

This architecture enables robust, scalable GitOps workflows with strong security boundaries and clear separation of concerns between components.