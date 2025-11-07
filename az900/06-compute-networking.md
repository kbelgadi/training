
# Compute

azure compute resources:
* vm (IaaS) -- equivalent to aws ec2 instance
* vmss (vm scale sets)
* app services (web apps)
* azure container instances (aci)
* azure kubernetes services (aks)
* windows virtual desktop

vm: renting a portion of a physical computer for your own use
    --> install whatever software you want, create/delete files...
    --> over 700 possibilities: combination of cpu, ram, disk size, iops

vmss: two or more vm running the same code
    --> vm added if usage is high --> auto scaling
    --> and vm removed if it usage slow down --> elastic service
1 vmss can handle 1000 vm

Azure app services: deploy code and conf --> azure will run it --> no access to machines --> PaaS

containers:
* container instances (aci)
* container apps: web services + features of k8s
* aks: 

azure virtual desktop: virtual desktop accessible every where you are (from IOS, android or any browser)


# Azure Functions

equivalent to aws lambda

small piece of code to run in the cloud
without need of a VM

triggerd on by:
* http call
* timer
* blob storage: new file => execute function
* message queue: new message => execute function
...

extremly cheap

usage examples:
* move files between blob storage containers on regular basis
* msg arrives in a queue => create 3 msgs in 3 other queues
* new blob uploaded in a storage => send email notification with attachment

# networking services

MS azure virtual network: network as software config (equuivalent to AWS VPC)

subnet: subdivision of virtual network (frontend, backend...)
VPN: connecting 2 networks as if they were the same network
express route: high speed connection to azure (ex: connect office to azure through expree route)
DNS: azure uses domain names (from registrar) to distribute trafic to the right location

protection services:
* ddos
* azure firewall: 
    --> restrict multiple virt networks in multiple subscriptions 
        -- managed network security service 
        -- builtin HA 
        -- unrestricted scalability
* network security groups (subnet and vm protection)
* private link 

topics not in the exam:
* LB
* app gateway
* cdn
* azure front door service: lb + cdn + fw (all in one)

monitoring
not in the exam:
* network watcher
* express route monitor
in the exam:
* azure monitor

# Network peering

to connect virtual networks
    --> console: virtual network --> peerings
    --> in the same region or across regions (global peering)
        --> additinal cost: outbound cost + inbound cost
        --> avoid hih volume of data

# Public / private endpoints

each resource created => who can access it and how

Example: storage account
* enable public access from all networks (including internet)
    --> always need an access key or storage account
* enable  public access from selected virtual networks (not the internet)
    --> need to enable storage endpoint on subnets (of allowed virtual networks)
* private access
    --> need to create private endpoint and private link to anyone who needs to access

all sort of resources have endpoints: sql dbs, cosmosdb, key vault...




