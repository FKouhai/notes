---
id: upgrade
aliases:
  - upgrade
tags:
  - kubernetes
---

# kubernetes_upgrade

- Update kubectl,kubeadm and kubelet
- Drain the cp node
- View the planned upgrade
- Apply the upgrade
- Uncordon the cp node
- Repeat the similar steps in worker nodes


## Commands

```bash
kubectl drain master --ignore-daemonsets
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.XY.Z
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon master

# On the worker nodes, instead of upgrade apply run
sudo kubeadm upgrade node
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon worker-node
```

![[Pasted image 20240430172815.png]]