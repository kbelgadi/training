# Design k8s cluster

## Questions to ask:
* purpose
education   --> minikube, single node k8s with kubeadm
dev/test    --> 1 master/multi-nodes, cloud managed (aks, eks...)
prod        --> HA multi-masters, kubeadm/GCP, kops/aws 
            --> check cloud provider recommandations on number of workers end resources
            --> storage: ssd (high pefs), network based (mltiple // connexions), persistent shared vols (shared between pods), label nodes for specific strage
            --> best practice: master for control plane, not for apps workloads --> add taint to master node
            --> separate etcd from master nodes
* cloud/on-prem
* workloads: how many, kind, resources requirements, traffic

## Choosing K8s infra

turnkey solutions: make it simple to deploy k8s on organizations
* openshift
* cloudfoundry container runtime
* VMWare PKS
* Vagrant

hosted solutions
* google gke
* openshift online
* azure aks
* aws eks

## K8s HA

controller manager multiple replicas:
            --> avoid duplicate actions
            --> leader election principle (param: --leader-elect=true)
            --> one of the replicas obtains a lease and becomes the leader during --leader-elect-lease-duration (default: 15s)
            --> the leader renews the lease every --leader-renew-deadline (default: 10s)
            --> all replicas tries to become leader every --leader-elect-retry-period (default: 2s) -- in case a control plane server crashes

kube scheduler follows the same approach (same params)

etcd as a k8s resource running on the control plane -- stacked topology
    --> simple to install/manage but risky during failure

etcd separated from the control plane master nodes -- external etcd topology
    --> harder to install/manage but less risky

kube api-server should point to the right etcd nodes --> param: --etcd-servers=https://hostname1:2379,https://hostname1:2379,...

## etcd HA

distributed, reliable key/value store
stores documents, each document is a map of key/value

deployable on multiple servers

consistency: same data on each node 
    --> how to manage concurrent writes of the same data: principle of leader/followers 
    --> read/write on the leader only 
    --> when a write comme on follower
            --> sent to the leader for processing
            --> the leader processes the write, 
            --> the leader ensures that the write is done on each follower 
            --> write complete only if leader receives consent from each follower

    leader election --> raft protocole
        --> after install, no leader
        --> timer init on each node
        --> the first to finish the timer, sends request to the others to become the leader
        --> the nodes receiving the requests send their vote  
        --> node assumes leader role
        --> leader sends notif to members to inform it is has the leader role
        --> no notificton received: reelection of leader initiated by survivors members
        --> concept of majority in raft: Qorum = N/2+1
        --> write is processed by the leader only if it received a consent from a majority (qorum) of nodes
        --> recommanded min of etcd nodes: 3 (choose an odd number: 3, 5, 7...)
        --> in case of infra distributed through 2 sites, take care to guaratee a qorum on at least one site 
                --> 6 nodes => qorum=4 => 4 nodes on siteA, 2 nodes on site B
                --> that's why it is better to have odd number of etcd nodes => to manage network segmentation

install: 
    --> download archives, 
    --> extract binary on `/usr/local/bin`
    --> generate certs
    --> run `/usr/local/bin/etcd`
            --> set --initial-cluster with the list of peers (other etcd nodes)

etcdctl to request etcd 
    --> ETCDCTL_API=3 => use API version 3 (default: API v.2)

## K8s install - kubeadm way

1. install container runtime (containerd)
    --> IPv4 forwarding
    --> install container engine
    --> configure cgroup driver (systemd): if containerd is using systemd driver, then same for kubelet (default systemd)
            --> verify init system (ps -p 1)
2. install kubeadm, kubelet and kubectl
3. init controlplane
    --> choose the pod cidr to pass to cluster init command
    --> choose the api server IP address to advertise workers
    --> execute: kubeadm init 
    --> set `~/.kube/config` file
4. Install networking addon (weavnet)
    --> deployed as demon set
    --> if pod cidr set when init cluster, then config the net driver (weave net) with IPALLOC_RANGE env var
5. join the workers to their master 
    --> use the command displayed during cluster init
