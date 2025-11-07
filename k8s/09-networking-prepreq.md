
# Switching and Routing 

add an ip addr to an interface
    --> `ip a add 192.168.1.10/24 dev eth0`

add a route:
    --> `ip route add 192.168.1.0/24 via 192.168.1.1`

conf linux to forward tcp packets between interfaces
    --> `echo "1" > /proc/sys/net/ipv4/ip_forward`

persist forwarding
    --> on `/etc/sysctl.conf`: `net.ipv4.ip_forward=1`

# DNS

dns conf
    --> on `/etc/resolv.conf`: `nameserver    192.168.1.100`

if same name entry on DNS and /etc/hosts
    --> default: host looks locally first to resolv the name, then look on the dns server if no resolution found localy

to modify the resolv order
    --> in `/etc/nsswitch.conf`: `hosts: files dns`

possible to conf dns server to forward dns request to external dns server (ex: 8.8.8.8)

www.google.com
    --> .       : root
        .com    : top level domain
        google  : domain name assigned to Google company
        www     : subdomain
            --> other sub domains: maps, drive, mail ...

DNS request steps to resolv app.google.com name: 
    internal organization dns server 
    --> internet: root dns --> .com dns server --> Google dns server which return IP --> back to internal org dns which store IP in cache for few seconds

to resolv web as web.company.com
    --> in `/etc/resolv.conf`: `search company.com` 
    --> ping web: the host will complete web with .company.com ang finaly ping web.company.com

DNS record types:
* A: map name to IPv4
* AAAA: map name to IPv6
* CNAME: map one name to another name

to check name resolution: 
* `nslookup wwww.google.com`
   NB: nslookup only queries the dns server, it does check local name resolution on /etc/hosts
* `dig wwww.google.com`

# Network namespaces in Linux

**namespaces**

create a network namespace named "red" in linux:
`ip netns add red`

list interfaces in created net ns:
`ip netns exec red ip link` 
or:
`ip -n red ip link` 
    --> this will show only "lo" (loopback) interface

list arp table in ns:
`ip netns exec red arp`

list routing table:
`ip netns exec red route`

**namespaces pipe**

to make two net ns (red and blue) communicate 
    --> create a pipe (virtual cable) between tho virtual interaces:
`ip link add veth-red type veth peer name veth-blue`
then attach a virt interfaces to each ns:
`ip link set veth-red netns red`
`ip link set veth-blue netns blue`

Assign an IP address to each virt interface:
`ip -n red add add 192.168.15.1 dev veth-red`
`ip -n blue add add 192.168.15.2 dev veth-blue`

Bring up the interfaces:
`ip -n red link set veth-red up`
`ip -n blue link set veth-blue up`

Then net ns can ping each others:
`ip netns exec red ping 192.168.15.2`

host cannot reach containers and containers cannot reach host

**bridge network**

to connect multiple ns 
    --> create a network 
    --> for that we need a virtul switch within the host
    --> openvswitch, linux bidge

create an interface on the host of type bridge:
`ip link add v-net-0 type bridge`

set it up:
`ip link set dev v-net-0 up`

before to connect ns on the host virtual interface
    --> delete previously created ns virtual interface:
`ip -n red link del veth-red`
`ip -n red link del veth-blue`

create a pipe to connect ns to bridge:
`ip link add veth-red type veth peer name veth-red-br`
`ip link add veth-blue type veth peer name veth-blue-br`

attach each first end interface to a ns:
`ip link set veth-red netns red`
`ip link set veth-blue netns blue`

attach the other end to the bridge network:
`ip link set veth-red-br master v-net-0`
`ip link set veth-blue-br master v-net-0`

add an ip address to each ns interface:
`ip -n red add add 192.168.15.1 dev veth-red`
`ip -n blue add add 192.168.15.2 dev veth-blue`

bring up the interfaces:
`ip -n red link set veth-red up`
`ip -n blue link set veth-blue up`

to reach ns interfaces from host:
    --> add ip addr to bridge interface
`ip addr add 192.168.15.5/24 dev v-net-0`

NB: this bridge interface is local to the host
    --> unable to reach the outside or from the outside to reach the bridge network

**from net ns to outside host**

To reach an external host 192.168.1.3
    --> add a route in the network ns with a gateway set to the bridge interface v-net-0
`ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5`

configure an iptables NAT rule on the host to replace the bridge "from" addr by the host addr:
`iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE`

Then, it will be possible to ping an external host from within a net ns:
`ip netns exec blue ping 192.168.1.3`

To reach the internet from network ns
    --> add a default gw as the host bridge ip address
`ip netns exec blue ip route add default via 192.168.15.5`

Then, we can ping 8.8.8.8 from within net ns:
`ip netns exec blue ping 8.8.8.8`

**reaching net ns from outside**

add a rule to tell:
    --> every trafic comming on host at port 80, should be forwarded to netns ip addr at port 80
`iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2 -j DNAT`

# Docker networking

container isolated from host and other containers:
`docker run --network none nginx`

container with no isolation with its host:
`docker run --network host nginx`

containers attached to a bridge network with default address 172.17.0.0/16
`docker run nginx`
    --> default name of the docker bridge network: `docker0`
    --> created under the hood by docker by running: `docker link add docker0 type bridge`

to list docker networks:
`docker network list`
    --> visible among host network interfaces

run container 
    --> docker create a net ns
        --> to see the net ns ID:
            `ip netns`
            or:
            `docker inspect <container ID>` # in `NetworkSettings` key
    --> docker create a paire of interfaces connected by a pipe, one interface attached to the bridge, the other attached to the container net namespace
        --> to see interface attached to the bridge network: 
            `ip link`
                    --> to see interface attached to the container: 
            `ip -n <docker container net ns ID> link`
                    --> to see IP addr assigned to the net ns interface of the container:
            `ip -n <docker container net ns ID> ip addr`

possible to access the container from:
* other containers attached to the same bridge network
* the host using the IP addr assigned to the container net ns

to see container from outside 
    --> docker port mapping:
`docker run -p 8080:80 nginx`
    --> now possible to access the container from outside using the container host IP addr and the mapping port (8080 as above)
    --> to do port mapping, docker creates iptables nat rule:
`iptables -t nat -A DOCKER -J DNAT --dport 8080 --to-destination 172.17.0.3:80`

    --> to see iptables rules docker creates:
`iptables -nvL -t nat`

# CNI (Container Networking Interface)

network namespaces work the following:
1. create net ns
2. create bridge network
3. create veth pairs (pipe)
4. attach veth to bridge network
5. attach other veth to bridge network
6. assign IP addr 
7. bring interface up
8. enable NAT - IP masquerade

same process for containers engines or container orchestration techs: docker, rkt, mesos, k8s

--> CNI: set of responsibilities for containers runtimes and plugins (programs that manage net ns for containers)

Example:

container runtime must:
* create net ns, 
* identify net the container to attach to, 
* invoke net plugin when container added/deleted, 
* specify net conf on container in JSON
...
plugin must:
* support add/del/check cmmands from cont runtime
* return result in particular format
...

CNI supported plugins: bridge, vlan, ipvlan, macvlan, windows, dhcp, host-local
plugins from third-party organizations: weave, flannel, cilium, vmware nsx, calico

Docker does not implement CNI (it implements CNM: container network model)

to use specific  net plugin (ex: cni-bridge with docker):
1. create a container without net ns
docker run --network=none nginx
2. invoke the bridge plugin:
bridge add <cont ID> /var/run/netns/<cont ID>

k8s creates docker container in none network
then invoke the configured CNI plugin







