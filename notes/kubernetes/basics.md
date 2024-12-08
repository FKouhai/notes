---
id: basics
aliases:
  - basics
tags:
  - kubernetes
---

# kubernetes_basics
K8S is a decoupled system with several objects(microservices) to manage the environment.

The communication between these microservices is API driven, the cluster configuration is stored as json in the etcd database, but its usually written in YAML, the kubernetes agents convert it to json so the information can be stored in the db

# COMPONENTS

- etcd
- kube-controlller-manager
- kube-scheduler → gets the pod spec for running containers coming to the API and places it on a node
- kubelet → Receives a requests to run the containers, manages resources and works with the container engine
- kube-proxy → Creates and manages networking rules to expose the container on the network to other containers or to the outside world
- cloud-controller-manager(on for cloud)
- containerd
- iptables/eBPF
- control-plane nodes → kube-api-server,scheduler
- worker or minion nodes
- kubectl → local client to communicate with the kube-api
- pod → Consists on 1 or more containers that share an IP, storage access and namespace, usually 1 container in a pod runs an application and the others support the primary application
- namespaces → Segregation of resources, used to keep objects differentiated from one another for resource control and multi-tenant considerations

### Orchestration

---

Managed through a series of watch loops, called controllers or operators, each controller queries the kube-apiserver for a particular object state, then modifies the object until the declared state matches the current state, the controllers are compiled into the kube-controller-manager but others can be added using CRDs.

The default and feature-filled operator for containers is a Deployment, it does not work with pods directly, but it manages ReplicaSets which is an operator that creates or terminates pods according to a podSpec that is sent to the kubelet which then interacts with the container engine to download and make available the required resources, then spawn or terminates containers until the status matches the spec.

The svc operator requests existing IP addresses and info from the endpoint operator and will manage the network connectivity based on labels, the svc is used to communicate between pods, ns and outside the cluster, there are also Jobs and CronJobs to handle single or recurring tasks among other def operators.

To manage several pods across different nodes we can use lables which are arbitrary strings that become part of the object mdata, nodes can have taints to discourage pod assignment unless the Pod has a toleration in its metadata

Annotations stay with the object but is not used as a selector
