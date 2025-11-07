## Rolling updates - Rollback

deployment create --> trigger a new rollout --> deployment new revision (ex: rev 1)\
new version of deployment --> new rollout --> deploy rev (ex: rev 2)\

Aim: track deploy revs and rollback if necessary

rollout status:
```
kubectl rollout status deploy/deploy_name
kubectl rollout history deploy/deploy_name
```

Default update strategy: rolling update
the pods are replaced the one after the other to not interrupt the app

to update the image:
```
kubectl set image deployment/my_deploy ...
```
But the update is not reflected on he manifest

deployment upgrade (following rolling update strategy):
 --> new replicaset created under the hood --> new pods attached to the new replicaset the one after the other --> old pods down the one after the other while new pods up


rollback: 
--> pods deleted on new replicaset --> pods started on old replicaset the one after the other while the old pods up the one after the other
```
kubectl rollout undo deployment/my-deployment
```

## Commands

A container lives as long as the process in it is running. Once the process stops, the container exits.

The process is launched by the command specified in CMD section of Docker file: `CMD sleep 100`  or `CMD ["sleep", "100"]` \
in JSON array format: CMD ["executable", "param1", "param2"]


possible to change the command during docker run:
```
docker run --name my_cont ubuntu sleep 100
```

To pass parameter to command in container, use ENTRYPOINT in Dockerfile:
ENTRYPOINT ["sleep"] 
    --> ENTRYPOINT ["<executable>"]
CMD ["120"] 
    ---> CMD [<list of parameters of the executable specified in ENTRYPOINT seperated by ",">]

Now, possible to override the sleep parameter value during docker run:
```
docker run --name my_cont ubuntu 100
```

to modify the entrypoint during docker run:
```
docker run --name my_cont --entrypoint sleep2.0 ubuntu 100
```

## Commands & arguments

In the previous example, to run a docker container with an executable and arg, we use both ENTRYPOINT and CMD in Dockerfile\

In a pod manifest, we use command and args:
```
apiVersion: v1
kind: Pod
metadata:   
  name: mypod
  labels: 
    env: dev
spec:
  containers:
  - name: mycontainer
    image: ubuntu
    command: ["sleep"]
    args:
    - "100"
```

`command` in pod def    --> overrides ENTRYPONIT in Dockerfile
`args` in pod def       --> overrides CMD in Dockerfile

## Environment variables

To set env var:
* into a pod, use `env` an array of (name, value) in the Pod/spec.containers.env 
```
...
  containers:
  - name: 
    image
    env:
      name: APP_COLOR
      value: blue
```


* from configMap, use `envFrom`:
```
...
  containers:
  - name: 
    image
    envFrom:
      configMapRef:
        name: myconfigmap
```

One env var at a time from configMap
```
...
  containers:
  - name: 
    image: 
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: myconfigmap
          key: APP_COLOR
```

create a config map:

* **imperative approach** from literal or file:
  using literal:
  ```
  kubectl create configmap myconfigmap --from-literal=APP_COLOR=blue --from-literal=APP_MOD=prod ...
  ```  
  using file: 
  ```
  cat << EOF >> app-config.properties
  APP_COLOR=blue
  APP_MOD=prod
  EOF

  kubectl create configmap myconfigmap --from-file=app-config.properties
  ```  

* **declarative approach** from manifest:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
  labels:
    env: prod
data:
  APP_COLOR=blue
  APP_MOD=prod
```

## Secrets

* **imperative approach** from literal or file:

  using literal:
  ```
  kubectl create secret mysecret --from-literal=USERNAME=user1 --from-literal=PASSWORD=xxxx  ...
  ```  
  using file: 
  ```
  cat << EOF >> app-secret.properties
  USERNAME=user1
  PASSWORD=xxxx
  EOF

  kubectl create configmap myconfigmap --from-file=app-secret.properties
  ```  

* **declarative approach** from manifest:

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  labels:
    env: prod
data:
  USERNAME=<base64 encoded value of user1>
  PASSWORD=<base64 encoded value of xxxx>
```

to code in base64: `echo -n "user1" | base64`

to see the Secret resource encoded data:

```
kubectl get secret mysecret -o yaml
```

to decode from base64: `echo -n "HJL#hg" | base64 --decode`

Usage within a pod manifest:
```
...
spec
  containers:
  - name: mycont
    image: myimage
    envFrom:
      secretRef:
        name: mysecret
```        

Or from secret by specifying a key:
```
...
spec
  containers:
  - name: mycont
    image: myimage
    env:
    - name: login
      valueFrom
        secretKeyRef:
          name: mysecret
          key: username
```        

Or from a volume:
```
...
spec
  volumes:
  - name: secret-vol
    secret:
      secretName: mysecret
```

kubernetes secret handling:
* A secret is sent to a node only if a pod on that node requires it.
* Kubelet stores the secret into a tmpfs so that the secret is not written to disk storage.
* Once the Pod that depends on the secret is deleted, kubelet will delete its local copy of the secret data as well.

A better way to secure password: helm secrets, hashicorp vault

## Multiple containers pods

containers on the same pod:
* share the same network space (refer to each others by localhost)
* access to the same storage volumes

no need for volume sharing or services.

on manifest, add a new container def under pod/spec.containers

all containers should be running, otherwise, the pod restarts undefinitely.

## Init containers

To start a task before the pod's containers start.\
Initcontainer = container definitions in Pod/spec.initContainers\
containers in initContainers are executed squentially the one after the other