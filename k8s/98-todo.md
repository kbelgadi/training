rollout 
    record
secret
    types
multicontainers
    shared volumes
maintenance
    upgrade: master + orker 
    etcd backup/restore: as service and as pod
multiple-scheduler
    --lock-object-name ?
security
    certificates and their paths for each component
    openssl basics: 
        cert display, 
        csr create for new user
    .kube/confi contexts
    api VS apis (cf. api ref in doc)
    rolebinding (matching user/role)
    create sa + pod --> check sa secret
storage
    pv, pvc types and states 
networking
    weave conf args and files on kubelet
    explore kube-proxy forwarding rules via iptables
    codeDNS debug (via busybox?)
    ingress rules: by host, by path