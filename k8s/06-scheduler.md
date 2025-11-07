# Scheduling

## Manual scheduling

Pod includes a nodeName properties not set by default\
no scheduler in k8s cluster --> schedule manually by setting nodeName\
if scheduler existes, it looks to all pods, and executes scheduling algo only on pod without nodeName property\
Set nodeName only on creation

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: mypod
       image: busybox
  nodeName:
     node01
```

On runtime --> create binding object in yaml, convert in JSON and pass it as data field by POST request to kube-apiserver pod's binding API:

```
apiVersion: v1
kind: Binding
metadata:
  name: mybindobject
target:
  apiVersion:
  kind: Node
  name: node01
```

curl --header ... --request POST --data { "apiVersion": "Binding"  } http://$SERVER/api/v1/namespace/default/pods/$PODNAME/binding
 
 Tip:\
 pod delete create --> kubectl replace --force\
 to watch pod in-progress deployment -> kubectl get po --watch

## Labels and Selectors

labels --> group resources by matter: app: app01, app: app02, tier: frontend, tier: backend, function: auth, etc
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    app: app01
    tier: frontend
spec:
  ...
```

Select pod with label:
```kubectl get po --selector app=app01```

replicaset/spec.template.metadata.labels = pods/metadata.labels != replicaset/metadata.labels

connect replicaset/pods --> replicaset/spec.selector.matchLabels = pod/metadata.labels

connect service/pods --> service/spec.selector = pod/metadata.labels

annotations --> record details for informatery purpose

## Taints and toleration

taint --> on node --> key=value:effect 
 
taint effect
* NoSchedule: not scheduled if no toleration to the taint
* PreferNoSchedule: no garanty that not scheduled if no toleration to the taint
* NoExecute: not scheduled and existing pod without tolerations are evicted

taint a node:
kubectl taint nodes node01 app=blue:NoSchedule

toleration on pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: app1
  labels:
    tier: frontend
spec:
  containers
  - name: app1
    image: busybox
  tolerations:
    key: "app"
    operator: "="
    value: "blue"
    effect: "NoSchedule"
```

toleration => pod may be scheduled on a node without any taint

install k8s with kubeadm --> master nodes tainted with NoSchedule effect to avoid running pods other that orchestration pods.

## Node selector

Define a label on a node:
kubectl label node key=value 

Then affect a pod to a node using nodeSelector property:
```
apiVersion: v1
kind: Pod
metadata:
    name: mypod
spec:
    containers:
    - name: mypod
      image: busybox
    nodeSelector:
      key: value
```

## Node affinity

Possibilty to use an operator and to specify behavior during scheduling or execution. 

spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In   
            values:
            - large
            - medium

Node affinity types:
* requiredDuringSchedulingIgnoredDuringExecution
* preferedDuringSchedulingIgnoredDuringExecution
* requiredDuringSchedulingrequiredDuringExecution

requiredDuringScheduling --> if no match, pod not scheduled anywhere
requiredDuringExecution --> evicted if the node label updated and does not match the label

## resource requirements and limits

The schedulers checks the node available resources before to place pod\
If not enough resources in any node, pod in state pending\
Default pod resources:
* 0.5 CPU
* 256 MiB RAM

If more resource necessary for a pod, add section `resources` in pod/spec.containers (a section for each container)

```
    resources:
      requests:
        memory: "512Mi"
        cpu: 1
```

CPU expression: 0.1 ou "100m" (milli-CPU) 
  --> 0.1 is the min value of k8s CPU res
  --> 1 CPU = AWS 1 vCPU, 1 GCP Core, 1 Azure Core, 1 Hyperthread
  --> 1 Gi = 1024 Mi = 1024 * 1024 Ki , 1G = 1000 M = 1 000 000 K

Default k8s limits:
* CPU: 1vCPU
* RAM: 512 Mi

A container can't use more CPU than limit

## Daemonset

A pod on each node\
when new node --> new instance of the daemonset/pod deployed on it\
use cases: monitoring, logs viewer, kube-proxy, etc\
in k8s until 1.12, nodeSelector added to each daemonset/pod\
after k8s 1.12, nodeAffinity and default scheduler\

## Static pods

pods created by the kubelet independently from the kube-apiserver\
it concerns only pods not replicaSet or deployment or services
* if the app crashes, the kubelet recreate the pod, 
* if manifest update, the pod is updated, 
* if manifest delete, the pod is deleted 

put the pod manifest in a dir\
indicate the dir as:
* the kubelet service arg: --pod-manifest-path 
* or (approach of kubeadm) --config=kubeconfig.yaml and in kubeconfig.yaml specify a key/value: staticPodPath: <pod manifest path> 

to show running pods: docker ps (kubectl needs kube-apiserver)\
kubelet can continue creating pods from kube-apiserver request\
kubeapi-server is aware of static pods (using kubectl get pods)\
kubelet create a mirror readonly object in kube-apiserver --- kubectl can list/read static pod but not update/delete static pod 

use case of static pod:\
deploy control plane pods as static pods -- create kubelet service, then put control plane manifests in pod-manifest-path 

kube-scheduler has effect on static pods

## Multiple schedulers

default scheduler: algo to distribute pod on nodes + conditions taints/toleration, nodeSelector, etc

use case: pod placement on node after additional checks

kube-scheduler service arg: --scheduler-name

kubeadm: scheduler as a pod
custom scheduler:
--leader-elect=true if scheduler deployed on multiple masters (to choose only one scheduler instance)
--lock-object-name=<custom scheduler name> to differenciate scheduler during leader election algo

pod manifest: specify the scheduler name as Pod/spec.schedulerName

to check if pod scheduled by the right scheduler: kubectl get events