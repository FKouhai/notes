---
id: etcd_backup
aliases:
  - etcd_backup
tags:
  - kubernetes
---

# etcd_backup

```bash
kubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl endpoint health"
# The previous command will output the health status
kubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl --endpoints=https://127.0.0.1:2379 member list -w table"
# The previous command will output in a table form the members of the etcd cluster
kubectl -n kube-system exec -it etcd-master -- sh -c "ETCDCTL_API=3 ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key etcdctl --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db"
# The previous command will create a backup/snapshot of the db
# the files on the host are located in /var/lib/etcd
mkdir backup
sudo cp /var/lib/etcd/snapshot.db $HOME/backup/snapshot.db-$(date +%m-%d-%y)
sudo cp /root/kubeadm-config.yaml $HOME/backup/
sudo cp -r /etc/kubernetes/pki/etcd/ backup/
# The previous commands will copy the important files to the backup folder
```

