

replicaset selector/mathclabels should match metadata/labels of pod template

To check api version and yaml sections of replicaset:
`kubectl explain replicaset`

Deployment manifest equivalent to replicaset except kind: Deployment

Generate Deployment manifest, then complte:
`kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3 --dry-run=client -o yaml > dpl.yaml`

All resources deployed in dfault ns:
`kubectl get all`

<service>.<namespace>.svc.cluster.local

Set current ns:
`kubectl config set-context $(kubectl config currentcontext) --namespace=dev`

endpoints: the pods that the service identified by matching service/spec.selector to pod/metadata.labels

Service of type NodePort:
(node - node ip) NodePort (restricted range: 30000 .. 32767) --> (service - clusterIP) port --> (pod - pod ip) TargetPort

service/spec.selector = pod/metadata.labels

to expose an existing pod through a service (if nodePort, no wayto specify the nodePort value):
`kubectl expose pod ...`

to create a service (no way to specify a selector):
`kubectl create service ...`

Pod includes a nodeName properties not set by default

`kubectl get po --watch`

`kubectl get po --selector app=app01`

replicaset/spec.template.metadata.labels = pods/metadata.labels != replicaset/metadata.labels

connect replicaset/pods --> replicaset/spec.selector.matchLabels = pod/metadata.labels

connect service/pods --> service/spec.selector = pod/metadata.labels

node taint => pod/spec.tolerations
node label => pod/spec.nodeSelector

Default pod resources:
* 0.5 CPU
* 256 MiB RAM


Status of a deployment rollout:
`kubectl rollout status deployment/nginx-deployment`

History of a deployment:
`kubectl rollout history deployment/nginx-deployment`

Annotate a deployment:
`kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"`

Rollback deployment:
`kubectl rollout undo deployment/nginx-deployment`

when you roll back to an earlier revision, only the Deployment's Pod template part is rolled back

Possible to pause rollout `kubectl rollout pause ...`, 
    do multiple changes, 
    then resume rollout `kubectl rollout resume ...` to execute the updates done during pause period

Rollback deployment:
`kubectl rollout undo deployment/nginx-deployment`

Dockerfile: ENTRYPOINT + CMD <=> Pod: command + args
args always in string type (not numbers)

`command` in pod def    --> overrides ENTRYPONIT in Dockerfile
`args` in pod def       --> overrides CMD in Dockerfile

the drained pods do not fall-back to the uncordoned node automatically

cordon node --> if pod running on it and an instance of replicaset, then pod continue working on it
            --> if pod running on it and an not instance of replicaset, not possible to cordon node --> need to delete pod (save its manifest, before)

etcd has its own CA --> kubeapi-server tls parameters to connect to etcd should indicate etcd CA (not kubeapi-server) CA

list of namespaced resources: `kubectl api-resources --namespaced=true`

docker run  --> "container layer" created on top of image layer
            --> files on images layers modified (that is readonly) --> copied first to container layer before modification
use `docker run ---mount` instead of `docker run -v...`

transient object: data created are deleted after the object has been released

persistentvolume of type hostPath 
    --> no storaeClass
        spec.hostPath.path
