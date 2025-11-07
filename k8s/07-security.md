# Security primitives

secure hosts
disable: root access, password based auth
only available: ssh key based auth

secure k8s
kube-apiserver
who can access --> auth mecanism: user/passwd in files, certificates, external ldap, machines service account\
what can do --> RBAC, ABAC (attributes based...), node authriz, webhook, etc

TLS encryption for comm kube-apiserver and:
* etcd
* kubelet
* kube-proxy
* kube-controlermanager
* kube-scheduler

between apps in the cluster:
network policies

# Authentication

k8s: nodes (physical, virtual), users (admins, devs, bots), end-users 

end-users: access secured by the apps --> out of course scope

users:
* humans --> impossible to declare users in k8s
* non-humans (service accounts) --> possible to create service accounts --> **not part of CKA but CKAD**

user access (kubectl or curl) managed by kube-apiserver

kube-apiserver auth mecanisms:

* static password file (not recommended)
csv file: `password,user,userid,group` (group column is optional)\
passed as arg to kube-apiserver service: `--basic-auth-file=user-details.csv` --> volume mount on kube-apiserver static pod \
request kube-apiserver by curl passing usr/password: `curl -v -k https://master-node:6443/api/v1/pods -u "user1/password1"`\
need to setup RBAC for new users

* static token file (not recommended)
csv file: `token,user,userid,group` (group column is optional)\
passed as arg to kube-apiserver service: `--token-auth-file=user-details.csv` --> volume mount on kube-apiserver static pod \
reload+restart kube-apiserver\
request kube-apiserver by curl passing token as bearer token: `curl -v -k https://master-node:6443/api/v1/pods  --header "Authorization: Bearer <token>"`\
need to setup RBAC for new users

* certificates

* 3rd-party auth protocol (ldap, kerberos, openid connect)

# TLS Basics

certificates/keys files naming conventions:
* certs (public key): `*.crt`, `*.pem` 
* keys (private key): `*.key`, `*-key.pem`

# TLS in k8s

3 kinds of TLS certs:
* server cert configured on server
* CA cert configured on CA
* client cert configured on client

k8s server certs:

* kube-apiserver: apiserver.crt, apiserver.key\
* etcd: etcdserver.crt, etcdserver.key
* kubelet: kubelet.crt, kubelet.key

k8s client certs to comm with kube-apiserver:

* admins: admin.crt, admin.key
* kube-scheduler: scheduler.crt, scheduler.key
* kube-controller-manager: controller-manager.crt, controller-manager.key
* kube-proxy: kube-proxy.crt, kube-proxy.key
* kube-apiserver can generate specific key pair:
  --> to talk to etcd, it can generate apiserver-etcd.crt, apiserver-etcd.key\
  --> to talk to kubelet, it can generate apiserver-kubelet.crt, apiserver-kubelet.key

CA : ca.crt, ca.key\
  --> may sign server/client certs and seperatly etcd

## Certs creation

tools: easyrsa, openssl, cfssl

***ca***
* priv key: `openssl genrsa -out ca.key 2048`
* csr: `openssl req -new -key ca.key -subj "/CN=kubernetes-ca" -out ca.csr`
* sign cert: `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt`

**client** admin user
* priv key: `openssl genrsa -out admin.key 2048`
* csr: `openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr`
* sign cert: `openssl x509 -req -in admin.csr -CA ca.key -CAkey ca.key -out admin.crt`

NB: 
* admin user --> csr subj includes O=system:masters
* for other clients: scheduler (O=system:kube-scheduler), controller-manager (O=system:kube-controller-manager), proxy (kube-proxy)

request kube-apiserver:
* curl
curl https://kube-apiserver:6443/api/v1/pods --key=admin.key --cert=admin.crt --cacert=ca.crt
* `kube-config.yaml`

**etcd**
generate pubic/priv key 
--> + generate peer public/priv key and peer ca cert on each etcd cluster node

**kube-apiserver**
since may be called as kubernetes, kubernetes.default, kubernetes.default.svc, etc \
    --> specify alt names in a `openssl.cnf` file \
    --> pass the cnf file to openssl req command when generating the csr: 
          `openssl req -new -key admin.key -subj "/CN=kube-apiserver" -out admin.csr --config openssl.cnf`

when running kube-apiserver, specify: 
* pub/priv/ca server keys
* pub/priv/ca client keys for etcd
* pub/priv/ca client keys for kubelet

**kubelet**
certificates named relatively to each node
cert files speficified in `kubelet-config.yaml` in each node

client cert to auth to kube-apiserver should be created with `O=system:node:node01` (for ex node01)

# Certs details

view certificates --> depends on the way the cluster has been installed:
* the hardway: kube-apiserver as a service
* kubeadm: kube-apiserver as a pod and manifest in `/etc/kubernetes/manifests/kube-apiserver.yaml`

View server cert details:
`openssl x509 -i /etc/kubernetes/pki/apiserver.crt -text -noout`
Check:
* Subject
* Suject Alternative Name
* issuer
* Validity (not after)

Investigate certs issues 
* cluster components installed as sys service: `journactl -u etcd.service -l`
* kubeadm installation: `kubectl logs etcd-master`
* if kube-apiserver is down, use docker CLI: `docker logs <etcd container ID>` or `crictl ps -a --no-trunc`

# Certs API

New commer 
  --> genarate a key + CSR 
  --> send csr to admin
Admin:
  --> use the CA to sign a cert from the csr

kubeadm install --> CA certs (pub/priv CA) stored in master node

k8s builtin cert API 
  --> send csr to k8s cert API via API call
Admin:
  --> create k8s object `CertificateSigningRequest`
      --> yaml manifest of kind `CertificateSigningRequest` including user csr encoded in base64 `cat jane.csr | base64 | tr -d "\n"`
  --> Then:
        view: `kubectl get csr -o yaml` --> to decode the csr in base64: echo "UHKJD..." `| base64 --decode`
        approve: `kubectl certificate approve jane`

**kube-controller-manager** is reponsible of managing certificates: 
  --> 2 controllers: csr-signing, csr-approving
  --> needs to start with args: `--cluster-signing-cert-file=/etc/...` `--cluster-signing-key-file=/etc/...` 

# KubeConfig

Request kube-apiserver 
* curl: `curl https://mycluster:6443/api/v1/pods --key admin.key --cert admin.crt --cacert ca.crt`
* kubectl: `kubectl get pods --server mycluster:6443 --client-key admin.key --client-certificate admin.crt --certificate-authority ca.crt`

--> move these conf args to KubeConfig file

default location of KubeConfig for kubectl: `~/.kube/config`

KubeConfig sections:
* clusters: contains `--server`, `--certificate-authority` 
* contexts: which user account to access which cluster *optionnally in which namespace*
* users: contains `--client-key`, `--client-certificate`
* current-context: default context

```
apiVersion:v1
kind: Config
current-context: 
clusters: []
contexts: []
users: []
```

View current context:
`kubectl config view`
`kubectl config view --kubeconfig=my-kubeconfig.yaml`

Change context (this will update `current-context` field in KubeConfig file):
`kubectl config use-context admin@production`

NB: 
* set certificates field with the full path to cert file
* possible to use the content in base64 of the certificate in the field `certificate-autority-data` of clusters section.

# API Groups

call to API:
* k8s version:  curl -k https://kube-master:6443/version
* list of pods:  curl -k https://kube-master:6443/api/v1/pods

API groups: (curl -k https://kube-master:6443/)
/metrics    /healtz  /version   /api    /apis  /logs

APIs responsible for the cluster features:

* /api... (core group)
  --> /v1/...{ namespaces, pods, rc, events, endpoints, pv, pvc, services, secrets, nodes, configmaps, secrets }

* /apis... (named group) 
  --> groups:
        /apps/...
        --> resources:
              /v1/{deployments, replicasets, statefulsets}/...
              --> actions (verbs):
                     list, get, create, delete, upate, watch
        /extensions
        /networking.k8s.io/v1/networkpolicies
        /storage.k8s.io
        /authentication.k8s.io
        /certificates.k8s.io/v1/certificatesigningrequest

k8s doc: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/

Access API via curl needs authentication:

* pass key, cert and ca in to curl cmde: curl -k https://kube-master:6443 --key admin.key --cert admin.crt --cacer ca.crt

* kubectl proxy (different than kube-proxy!): start a local http proxy service on port 8001 to ensure auth to kubeapi-server using creds in $KUBECONFIG file

# Authorization

Restrict access to users within their namespaces.

Autorization mechanisms

* Node
kubelet -----> kubeapi-server <----- user 
    read: services, endpoints, nodes, pods 
    write: node status, pod status, events
    
    kubelet auth to kubeapi-server via certificate with group: system:nodes and CN: system:node:node01
    
    ---> node authorizer (it checks if cert group: system:nodes and CN: system:node:node01)

* ABAC
  ex: dev user "alice" authorized to do anything on pods pod within dev namespace
  --> deploy Policy kind resource to kubeapi-server:
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", 
        "spec": {"user": "alice", "namespace": "dev", "resource": "pods", "apiGroup": "*"}}

* RBAC 
  instead of an authorization for each dev (ABAC approach), create a role for developers and associate all the devlopers to that role (RBAC approach).

* Webhook
  api call to external authorization service
  --> api call to third-party authorization service (ie: open policy agent) and let it repond if the user should be permitted or not.

* AlwaysAllow, AlwaysDeny
  By default, the authorization mecanism is: AllwaysAllow

**--authorization-mode**: 
  kubeapi-server parameter
  if a list: Node, RBAC, Webhook -- authorization processed by each authorizer in the order of the list until an authorizer grant the request
  by default: AlwaysAllow

# RBAC 

Create a role:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["list"]
```

NB: 
>apiGroups empty in case of core group API

Role binding:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

To check if you access to a resource/action:
`kubectl auth can-i create deployments`

As an admin, to test user's authorizations in specific namespace:
`kubectl auth can-i as jane create deployments --namespace test`

Role resourceNames:
role.rules[i].resourceNames: ["pod1", "pod2"]

# Cluster roles

resources:
* namespaced (pods, deployments, role/rolebindings...)
* cluster scoped (nodes, pv, certificatesigningrequests, namespaces, clusterroles, clusterrolebindings...)

to see full list of namespaced or non-namespaced resources:
`kubectl api-resources --namespaced=true`
`kubectl api-resources --namespaced=false`

Create a role:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

Create a Rolebinding:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

ClusterRole and ClusterRoleBinding --> no namespace specified
ClusterRole for namespaced resources --> access to resources in the whole cluster

# Service account

2 types of accounts: user (humans), service (applications)

examples of apps using service accounts: prometheus, k8s dashboard, etc

create sa:
`kubectl create serviceaccount dashboard-sa`
  --> token created stored as a secret named <sa name>-token-wxyz
  --> used as an authentication bearer token while making a curl REST call
  --> curl --insecure https://<kubeapi-server IP>:6443/api --header "Authorization: Bearer xxxxx..."

if sa for app deployed in k8s --> mount sa secret token as a volume to the app pod

each namespace has a default sa with authorization to run basic API queries
every time a pod is created in a namespace --> default sa token is mounted automatically as a volume to the pod
--> kubectl describe pod --> see default sa token volume mount point

to avoid associating default sa to a pod:
pod/spec.autoMountServiceAccountToken: false

Add a specific sa to a pod:
pod/spec.serviceAccountName: dashboard-sa

pod: no edit sa
deployment: edit sa --> rollout pods

**1.22 updates**
default token does not have expirity
KEP 1205 (Kubernetes Enhancement Proposal)
--> TokenRequestAPI to create a token audience bound, time bound and object bound
--> default sa --> defautl sa token mounted as a projected volume into the pod with expiration time

**1.24 updates**
KEP 2799

sa created `kubectl create sa dashboard-sa`
  --> no token created automatically 
  --> token should be created explicitely: `kubectl create token dashboard-sa` 
  --> token displayed to screen with expiry duration of 1 h since create time

possible to create a token as a secret automatically with sa
  --> specify sa name as annotation of token secret resource
      ```  
      apiVersion: v1
      kind: Secret 
      type: kubernetes.io/service-account-token
      metadata:
        name: mysecret
        annotations:
          kubernetes.io/service-account.name: dashboard-sa
      ```
        --> non expiring token associated to sa
        --> be sure that a non expiring persisting token is acceptable for you

# Images security

image:nginx --> docker.io/library/nginx 
                ------     -----  ----
                registry   user    docker repo
                      (default=   
                      library)  

other docker registry: gcr.io (google)

private registry:
login: `docker login private-reg.io`
docker run: `docker run login private-reg.io/internal-app`

create a secret of type `docker-registry` including docker private reg creds:
`kubectl create secret docker-registry regcreds private-reg-sec --docker-server=... --docker-user=... --docker-password=... --docker-email=...`

declare the secret in: `pod.spec.imagePullSecrets.name: regcreds`

# Security context

docker capabilities:
`docker run --cap-add MAC_ADMIN ubuntu`

Kubernetes security contexts: pod level or container level
pod level: `pod.spec.securityContext.runAsUser=100`
or container level: `pod.spec.containers[i].securityContext.runAsUser=100`

NB: capabilities are only supported at container level

# Network policies

ingress trafic: trafic coming
ergess trafic: trafic going out

NetworkPolicy example: only allow igress trafic from API pod on port 3306 
  --> block all other trafic from other pods 
  --> only allow networkpolicy specified trafic


Use labels to label pods
Use Selectors on NetworkPolicy to specify matched pods

Exemple of ingress policy on db pod:


DB Pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    role: api
spec:
...
```

NetworkPolicy example:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            role: api
      ports:
        - protocol: TCP
          port: 3306
```

Only ingress trafic affected 
  --> egress trafic unlimited

NetworkPolicy enforced and depend on network solution:
>flannel --> Do not support NetworkPolicy
  calico
  kube-router
  romana
  weavenet

NB:
If network solution does not support networkPolicy --> networkPolicy created but without effect