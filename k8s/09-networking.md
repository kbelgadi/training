# Cluster networking

kubeapi-server          --> port: 6443
kubelet                 --> port: 10250
kube-scheduler          --> port: 10251
kube-controllermanager  --> port: 10252
etcd server for clients --> port: 2379
etcd server for peers   --> port: 2380
services type nodePort  --> ports: 30000 to 32767

URL from k8s doc to download weave CNI plugin:
https://v1-22.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node

# Pod networking

each pod IP address
k8s does not come with pod networking solution out of the box: need to deploy one (weave, calico, flannel, cilium, vmware nsx...)

k8s nodes communicate in a physical network
each node its ip address
1. containers created --> k8s creates net ns for them
2. for containers to communicate 
        --> attach net ns to bridge net on each node 
        --> **each bridge net in a node has its own vlan ip range**
3. a pipe is created for each container 
4. a route is added in each node to reach net bridge via host physical net interface 
        --> to let the containers communicate with others between different net bridges (different nodes)

kubelet executes the steps above by executing the CNI script
    passed to kubelet:
    --> CNI conf: --cni-conf-dir
    --> CNI scripts: --cni-conf-bin

# CNI in k8s

CNI plugin is invoked by the component within k8s, that is responsible of creating components
The component must invoke the appropriate net plugin "after" the container is created

CNI plugin is configured in the kubelet service --network-plugin=cni --cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d
--cni-conf-dir kubelet will use the first one in alphabetic

weaveworks:
deploys agent on each node
each agent stores the topology of the entire setup (nodes, their ips, etc)
bridge created on each node and named `weave`
a pod may be attached to both docker bridge and weave bridge
each pod has a route configured on weave agent
when packet sent from pod1/node1 to pod3/node3: 
    --> weave agent intercept the packet and encapsulate the packet source-pod-ip/dest-pod-ip into a packet with source-node-ip/dest-node-ip 
    --> when received by node3, weave agent intercept the packet, decapsulate, and sends to pod3

weave is deployed as a daemonset on the cluster

# Weave IP address management

how is the bridge network assigned an IP subnet?
how is the pod assigned an IP?
where is the info stored and who is responsible of assigning IP addresses and managing that no duplicate IP assigned?

CNI plugin is responsible of that
list of IPs/pod assigned stored in a file

CNI builtin plugins: DHCP, host-local
to manage IP addresses from the host (file of IP lists) --> host-local plugin

CNI conf file (`/etc/cni/net.d/net-script.conf`) section `ipam` to the type of plugin (ex: host-local), the ip range (subnet) and the route

weave default ip range: 10.32.0.0/12 (~ 1M addresses)

# Services netwoking

k8s service: for pods to access each others wherever the node where they are hosted
    --> service hosted accross al nodes
    --> from within the cluster only --> service type: ClusterIP

to access a pod from outside --> service type: NodePort
    --> the service is exposed from all cluster nodes 

how k8s services works:
    kubelet instance in each node 
        kubelet listen kubeapi-server for pod creation/update 
        new pod --> kubelet intercep the update 
                --> creates the pod's containers 
                ---> and invoke CNI to attach the pod to the net bridge and assign ip address to container wthin bridge network
    kubeproxy instance in each node
        kubelet listen kubeapi-server for service creation/update
        service: cluster-wide concept, virtual object: no net ns, no bridge no interface attached to bridge, etc
        service created --> assignded IP address from predefined range
                        --> each kube-proxy instance in each node and service ip address
                        --> kube-proxy creates forwarding rule: each trafic comming to this ip address (ip:port combination) is forwarded to pod ip
                        --> when service deleted, kube-proxy deletes the rule
        forwarding rules created using: userspace, iptables or ipvs
            --> passed in kube-proxy args
        service ip range --> passed by args to kube-proxy
        pods and services ip range should not overlap: each has its dedicated subnet

    list of iptables NAT rules for service named `db-service`:
        `iptables -L -t nat | grep db-service`

    kube-proxy logs to see NAT rules creation (check verbosity of the kube-proxy service): 
        `cat /var/log/kube-proxy.log`

# DNS in k8s

k8s includes a builtin DNS service
service created --> k8s dns creates a record ("service name","service address")
                --> pod accesses another pod using its service name (within the same namespace)
                --> in separate ns: `<service-name>.<namespace>.svc.cluster.local` 
                    --> svc: services subdomain
                    --> cluster.local: root domain of pods and services

NB:
no dns records created for pods --> if needed, should be specified explicitely
                                --> pod dns name: `<pod ip replace "." by "-">.<namespace>.pod.cluster.local`

# coreDNS in k8s

in each pod 
    --> and entry in `/etc/resolv.conf` to indicate the DNS ip address

service or pod created 
    --> record added for the service or pod (pod dns name = replace "." by "-" in pod ip) in the DNS server

before k8s 1.12: dns service name --> kube-dns
after 1.12: recommended dns service --> coreDNS

coreDNS server  --> deployed within the cluster --> replicaset of 2 replicas
                --> coreDNS conf: `/etc/coredns/Corefile`
                    --> specifies plugins conf: handling errors, recording health, providing metrics...
                    --> `kubernetes` plugin: makes coreDNS working with k8s is: 
                        --> where k8s root domain is specified: cluster.local
                        --> includes options: 
                                pods option: to create dns records for pods
                    --> `proxy` plugin: where the dns server to forward name resolution is defined (in `/etc/resolv.conf`)
                        --> `/etc/resolv.conf`: deployed as config map

coreDNS pod deploy  --> service `kube-dns` created
                    --> service address is defined as the nameserver of the pod in its /etc/resolv.conf
                    --> responsible for that: kubelet

ip of cluster DNS server for k8s (service IP): configured in kubelet yaml config file (`/var/lib/kubelet/config.yaml`)

`search` option in /etc/resolv.conf: name suffixes that may be added to the name
    --> only available for k8s services
    --> for pods: full fqdn must be used

# Ingress

on cloud: nodePort service + LB --> 1 LB for each URI + umbrella LB --> too many LBs --> operations made difficult + increasing bill

ingress can replace (URLs path LBs + umbrella LB)

how to deploy?
    --> ingress controller: deployed a proxy solution: nginx, haproxy or traefic
    --> ingress resources:  configuration
 
ingress controller solutions: GCE (from GCP), nginx, contour, haproxy, treafic, istio
    --> maintained by k8s project: GCE, nginx

case of nginx ingress controller
    --> use configmap to decouple the controller conf from its image
        --> args:
              - /nginx-ingress-controller # ingress controller executable
              - --configmap=${POD_NAMESPACE}/nginx-conf
            env:
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.manespace

    --> ingress controller will expose ports. Ex: 80 and 443
        --> needs a service
    --> ingress controller will monitor k8s for ingress resources
        --> needs a service account

ingress resource
    --> specify service name + port
    --> rules: to route trafic dependin on host or URL path


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
spec:
  rules:
  - http:
      paths:
      - path: /wear
        backend:
          service:
            name: wear-service
            servicePort: 80
       - path: /watch
        backend:
          service:
            name: watch-service
            servicePort: 80
```

when ingress resource describe 
    --> we find: `default backend`: if user url does not match any rule --> user redirected to default backend

in case of multiple domains:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
spec:
  rules:
  - host: wear.online-shop.com
      http:
        paths:
        - backend:
            service:
                name: wear-service
                servicePort: 80
  - host: watch.online-shop.com
      http:
        paths:
        - backend:
            service:
                name: watch-service
                servicePort: 80
```

split trafic by url: 1 rule , 2 paths
split trafic by hostname: 2 rules , 1 path for each
