# OS Upgrade

Node unavaiblable --> pods evicted after 5m (by default)
    --> param: `--pod-eviction-timeout=5m0s`

empty the node from its pods:
```kubectl drain mynode```

mark the node as unschedulable:
```kubectl cordon mynode```

mark the pod as schedulable again
```kubectl uncordon mynode```

the drained pods do not fall-back to the uncordoned node automatically.

# K8s versions

kubernetes version 1.23.10 = major.minor.patch
* minor: every few months with new features, functionalities
* patches: often with critical bug fixes

v 1.0: jul 2015

alpha: features disabled and potentially buggy
beta: code well tested and features enabled

download tar ball:
* all control plane components (apiserver, controller-manager, scheduler, kubelet, proxy) and kubectl in the same version
* other components without the same version numbers: etcd, coreDNS

# Upgrade process

apiserver: primary component. Suppose at v. n:

none of the other control plane components can be at a higher version, but it may be at a lower version:
* controller-manager and scheduler could be at v. n-1
* kubelet and proxy could be at v. n-2
* kubectl could be at v. n-1, n, n+1

if v. n released --> supported versions: n, n-1, n-2

upgrade: one minor version at a time
from 1.10 to 1.13: 1.10 --> 1.11 --> 1.12 --> 1.13

upgrade process:
* cloud managed: simplified cloud provider upgrade feature
* kubeadm: `kubeadm upgrade plan && kubeadm upgrade apply`
* the hard way: manualy

upgrade process --> master node first 
                --> kubectl not working nor controller-manager or scheduler 
                --> applications on workers continue to work as normal 
                --> once masters upgraded, they come up and the cluster at it whole is working

workers upgrade strategies:
* all workers at once: applications down during upgrade
* upgrade the workers one after the other --> workloads are scheduled on other nodes during upgrade (cordon/drain on maintained node)
* on cloud infra: create new node with newer version, move workloads to the new, upgrade the old version node, etc

with kubeadm:
kubeadm upgrade plan 
upgrade kubeadm to n+1: `apt-get upgrade kubeadm=1.12.0-00`
upgrade control plane componnents to n+1
    --> `kubectl get node` shows version n on nodes because kubelet is not upgraded yet
upgrade kubelet manualy to n+1
    --> `kubectl drain node1`
    --> `apt-get upgrade kubeadm=1.12.0-00`
    --> `apt-get upgrade kubelet=1.12.0-00`
    --> `kubeadm upgrade node config --kubelet-version 1.12.0`
    --> `systemctl restart kubelet`
    --> `kubectl uncordon node1`

# Backup / Restore methods

backup candidates:
* resources configuration
* etcd
* volumes

**resources config**

declarative approach --> yaml manifests in github
imperative approach --> no files to backup

backup via kubectl (for few resource groups):
>```kubectl get all --all-namespaces -o yaml > all-resources.yaml```

or by querying kube-api server directly:
>Velero by Heptio (ex Ark)

**etcd backup**

etcd service arg `--data-dir`: where etcd data are stored --> you can backup the dir

use etcd builtin snapshot: ```ETCDCTL_API=3 etcd snapshot save etcd-backup.db```
backup status: ```ETCDCTL_API=3 etcd snapshot status etcd-backup.db```

restore proc:
1. stop kube-apiserver service
2. `ETCDCTL_API=3 etcd snapshot restore etcd-backup.db --data-dir /var/lib/etcd-from-backup`
3. **IMPORTANT**: if external etcd, `chown -R etcd:etcd /var/lib/etcd-from-backup`
3. configure etcd.service to use the new data dir by setting `--data-dir` arg
4. reload, restart etcd.service
5. start kube-apiserver service

NB: 
1. with etcdctl commands, declare etcd endpoints, cacert, cert and key
```
--endpoints=https://127.0.0.1:2379
--cacert=/etc/etcd/ca.crt
--cert=/etc/etcd/etcd-server.crt
--key=/etc/etcd/etcd-server.key
```
2. in managed cloud k8s, no access to etcd cluster, so backup via kube-apiserver
