
# Core concepts

## kubernetes architecture

master components: 
* kube scheduler
* kube controller manager
* kube apiserver
* etcd

worker:
* container runtime
* kubelet
* kube proxy

## etcd

key/value db
store k8s config data
launch procedure depends on k8s install approach: kubeadm or from-scratch
k8s from scratch: 
  --> download etcd binary, run as a service with multiple params
  --> advertise client url: current server / port:2379 -- configured on the kube-apiserver to reach etcd
k8s using kubeadm:
  --> etcd as a pod
  --> list k8s keys:
        kubectl -n kube-system exec etcd-master -- etcdctl get / --prefix --keys-only
etcdctl uses etcd api v2 (default), v3
specify etcd api version: export ETCDCTL_API=3
v2 commands examples:
* etcdctl backup
* etcdctl cluster-health
* etcdctl mk
* etcdctl mkdir
* etcdctl set
v3 commands examples:
* etcdctl snapshot save 
* etcdctl endpoint health
* etcdctl get
* etcdctl put

## Kube-api server

Kube-api server is the primary management component in kubernetes \
kube-api server responsible of: authentication, interraction with etcd \
kubectl --> kube-api server --> etcd \
OR: curl POST --> kube-api server --> etcd \
Pod creation: \
```kubectl apply -f ... ```	\
            --> kube-api request authentication and validation \
            --> kube-api server creates a pod not assigned to node \
            --> kube-api updates etcd \
            --> scheduler detects new pod and identify the right node \
            --> scheduler informs kube-api server \
            --> kube-api updates etcd \
            --> kube-api informs kubelet on the worker \
            --> kubelet creates the pod (containers) \
            --> kubelet informs api server about status \
            --> api server updates etcd 

kube-apiserver runtime options:
k8s installed by kubeadm:
  --> kube-apiserver as pod
  --> options stored in pod folder: `/etc/kubernetes/manifests/kube-apiserver.yaml` 
k8s installed from scratch:
  --> kube-apiserver as a service
  --> options stored in `/etc/systemd/system/kube-apiserver.service`

## Controller

A controller is a process that continuously monitors the state of various components within the system \
Node controller monitors nodes status and takes necessary actions: \
  --> node heartbeat check every 5s, \
      node marked unreachable 40s after stop receiving heartbeat, \
      after 5mn not comming back up, pods on node are removed and assigned to another node \
replication controller controls replica sets, a new pod is created if a pod dies so that it maintains the number of pods in the set\
other controllers: namespace controller, endpoint controller, service account controller, etc \
all conrollers packaged on a single process: kube-controller manager \
possible to enable/disable subset of controllers by modifying `controllers` option

## scheduler

scheduler is only responsible for deciding which pod goes on which node.\
It doesn’t actually place the pod on the nodes. That’s the job of the kubelet.\
The scheduler excludes the nodes that do not fit the profile for this pod (cpu/mem requests).\
On the candidate nodes, it uses a priority function to assign a score to the nodes on a scale of 0 to 10.\
For example the scheduler calculates the amount of resources that would be free on the nodes after placing, and up-rank the node with the more remaining resources.\
Scheduling may be constrainted by node selectors, affinity, taints/toleration.\
Possible to write your own scheduler\
install: pod if kubeadm install or service 

## Kubelet

kubelet loads or unloads containers on the worker as instructed by the scheduler on the master.\
it sends back reports at regular intervals on the status of the worker and the containers on them.\
it requests the container runtime engine, which may be Docker, to pull the required image and run an instance.\
kubelet monitors pods and containers and reports to kube-apiserver on a timely basis.\
If you use kubeadm tool to deploy your cluster, it does not automatically deploy the kubelet.\
You must always manually install the kubelet on your worker nodes. Download the installer, extract it and run it as a service.

## Kube-proxy

Kube-proxy is a process that runs on each node in the kubernetes cluster. \
every time a new service is created its kube-proxy creates the appropriate iptables rules on each node to forward traffic to those services to the backend pods. \
using iptables rules so that each trafic comming to service's IP is fowarded to pod's IP. \
kubeadm deploys kube-proxy as a deamon set \
(pods communicate with each others through services)

## Pod

pod yaml manifest
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
  labels:
      type: backend
spec:
  containers:
    - name: redis
      image: redis
```

without yaml maifest:
```
kubectl run redis --image=redis --namespace=finance
```

## Replication controller

Replication controller ensures that the specified number of pods are running at all times, even if it's just one or hundred\
multiple pods to share the load across them\
replication controller the older technology replaced by **replica set**

### replication conroller yaml

```
apiVersion: apps/v1
kind: ReplicationController
metadata:
    name: redis
    type: backend
spec:
    template:
        metadata:
            name: redis
            labels:
                type: backend
        spec:
            containers:
                - name: redis
                  image: redis
    replicas: 3
```

### replica set yaml

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: redis
    labels:
        type: backend-rs
spec:
    template:
        metadata:
            name: redis
            labels:
                type: backend
        spec:
            containers:
                - name: redis
                  image: redis
    replicas: 3
    selector:
        matchLabels:
            type: backend
```

Replica set needs "selector", it is mandatory \
In replication controller the selector is not mandatory

with selector, replication controller or replica set monitors other pods created previously with the specified label.
selector/mathclabels should match pod's metadata.labels

template def section is required to let the replica set replace the pod if it is destroyed

to increase pods replicas:
- update replicas atribute in the yaml manifest then: kubectl replace -f ...
- kubectl scale --replicas=6 -f rs-manifest.yaml
- kubectl scale --replicas=6 replicaset rs-name

To check api version and yaml sections of replicaset:
`kubectl explain replicaset`

## Deployment

higher in the hierarchy of: pod -- replica set -- deployment\
multiple instances of an application\
update app every time a new version is available\
update instances of app the one after the other to not disturb users --> rolling update\
rollback changes\
pose, update, resume for the pods to roll out together\
yaml manifest equivalent to replica set except kind: Deployment

use ```kubectl create``` with ```--dry-run=client -o yaml``` to generate yaml manifest:
```
kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3 --dry-run=client -o yaml > dpl.yaml
```

show all objects created in default namespace:
`kubectl get all`

## Namespace

Three namespaces created automatically by k8s: 
* default
* kube-system: internal k8s services isolated from user's resources
* kube-public: resources available to all users

on the same namespace, services should be called by its names\
between different namespaces, services names should be appended by the namespace: <service>.<namespace>.svc.cluster.local\
when service created, a dns entry is added automatically in this format:\
* cluster.local: default domain
* svc: service sub-domain

to get a resource within a namespace:
```
kubectl get pods --namespace=dev 
kubectl get pods -n dev 
```

To set the current namespace and avoid typing it every kubectl command:
```
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

to view resources on all namespaces:
```
kubectl get pods --all-namespaces 
```

create a namespace:
```
kubectl create namespace dev
```

namespace yaml manifest:
```
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

pod yaml manifest with namespace
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
  namespace: dev
  labels:
      type: backend
spec:
  containers:
    - name: redis
      image: redis
```

to limit resources by namespace, create a quota:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```
## Services

Allow comm between groups of apps, between users and apps and between app and external resources.

endpoints: the pods that the service identifies by matching service/spec.selector to pod/metadata.labels

**nodePod service**

listen request on a port on the node and forwards it to the port of the pod running the app\
(node - nde ip) NodePort (restricted range: 30000 .. 32767) --> (service - clusterIP) port --> (pod - pod ip) TargetPort 

```
apiVersion: v1
kind: Service
metadata:
  name: myservice
  namespace: dev
  labels:
      type: backend
spec:
  type: NodePort
  ports:
    - targetPorts: 80
      port: 80
      nodePort: 30008
  selector:
      app: redis
      type: datastore
```
if no targetPort specified --> assumed to be same as port \
if no nodePort specified --> a random port on the node is choosed in range 30000 .. 32767 \
if pods on multiple nodes --> service spans accross all nodes with the same node port on each node

service/spec.selector = pod/metadata.labels

multiple pods on single node --> load balace between target ports with session affinity\ 
multiple pods on mutiples nodes 
  --> service accross nodes 
  --> service port (NodePort) on each node 
  --> load balance between target ports with session affinity

to see the services:
```
kubectl get services
```

**cluster ip service**

default service type (if `service/spec.type` not specfied) \
create a virtual ip on the cluster and forward the request to a set of pods \
use labels to identifiy pods to load balance accross \
lb algo: random \
pods are accessible through attached ClusterIP service name

```
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: dev
  labels:
      type: backend
spec:
  type: ClusterIP
  ports:
    - targetPorts: 80
      port: 80
  selector:
      app: myapp
      type: backend
```

type: ClusterIP is the default type whenever type is not specified

**load balancer service**

use a lb service of a cloud provider to distribute requests across nodePorts on the cluster nodes

```
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: dev
  labels:
      type: backend
spec:
  type: LoadBalancer
  ports:
    - targetPorts: 80
      port: 80
      nodePort: 30008
  selector:
      app: myapp
      type: backend
```

## Imperative / declarative

**Imperative approach**

Create an NGINX Pod
```
kubectl run nginx --image=nginx
```

Generate POD Manifest YAML file (-o yaml). Don't create it (--dry-run)
```
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

Create a deployment
```
kubectl create deployment --image=nginx nginx --replicas=4
```

Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)
```
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

scale a deployment
```
kubectl scale deployment nginx --replicas=4
```

Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379
```
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```

This will not use the pods labels as selectors, instead it will assume selectors as app=redis, so generate a yaml file and modify it
```
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```

Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes
```
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml
```

This will not use the pods labels as selectors
```
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

Create a pod and its service in one command using run option `expose=true`:
```
kubectl run httpd --image=httpd:alpine --port=80 --expose=true
```

Tip:\
Use `kubectl expose` command.\
If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.\
Possibility: Use `kubectl create` command if NodePort service, but no way to specify a selector. So generate a manifest with --dry-run then add the selector.

**Declarative approach**

Create a deployment:
`kubectl create -f deploy.yaml`

Update a deployment
`kubectl replace -f deploy.yaml`

To avoid recording object versions:
`kubectl replace --force -f deploy.yaml`

`kubectl create ... ` error if the object exists\
`kubectl replace ... ` error if the object does not exist\
To avoid such errors, use `kubectl apply ...`



## kubectl apply

When kubect apply:
1. the yaml file is read
2. An object conf is created in k8s with added status fields: the live conf
3. The yaml version of the local object configuration file is converted to a JSON format, and stored as the last applied conf.\
   This last applied conf is stored as annotation in the metadata section of the yaml live conf (only when apply cmd)

When update the object, the three are compared:
if changes to the live conf --> apply the changes --> json format updated