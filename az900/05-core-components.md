# Regions, region pairs sovereign regions

actually azure 60+ regions in the world

regions = a set of datacenters (min 3)

every region has a pair region (fast connection between the two pairs)

useful when doing backup or massive tranfers

to duplicate your services --> choose the pair region

azure not doing rollout update on both regions pairs at the same time 

to explore global datacenters map: https://datacenters.microsoft.com/globe/explore

canada has two regions with data residency (data cannot leave canada regions, but anyone can create resources in these regions)

brazil has one region --> pairs with south central US --> oe way pairing (bakup to US only)

no region pairs for qatar region

create resource --> you can choose region --> pricing different

in some region  you have to be resident to choose it

sovereign region --> disconnected from azure public cloud --> compliance standards --> reserved to gov, military, secret agencies
* azure commercial: the public cloud
* azure gov
* azure gov secret
* azure gov top secret
* china: separate company that operate azure in china --> you need to be on a company that has business in china

# AZ

AZ
* 1 DC or a set of DC with high speed connections in between
* separate location within each region
* separate buildings

if need to use AZ --> choose a region with AZ (not all regions have AZ)

not every service supports the concept of AZ

3 types of AZ services:
* zonal services: choose an AZ to deploy a servise --> then deploy a duplicate service in another AZ --> ex: VM
* zone-redundant: service automatically deployed through AZs (no need to configure AZ) --> ex: azure sqldb, azure cosmosdb
* always vailable: global services --> non regional --> always available --> azure portal, AD, front door

some services gives you choice between zonal/zone-redundant:
* lb
* app gateway

# Resources & Resource groups

resource: accssible azure service (vm, storage account, db...)
    --> create using azure portal, cli, ps, arm template...
    --> has a name (advice: use naming convention) --> sometimes unique in azure, sometimes unique only in your resource group
    --> general: indicate region where to deploy
    --> in azure portal --> tab --> all resources: all your running resources
    --> most resources have cost
    --> 1 resource --> associated to 1 subscription (subscription: where cost get billed)


resource group:
    --> sort of folder or container, grouping resources
    --> belongs to a region --> but may contain resources from diff regions
    --> MS recommend that res in res group have connection between each others --> do not put disparate res
    --> 1 res belongs to 1 res group
    --> permissions can be assigned as res group level
    --> no security boundary (ex:firewalling) offered by res group

# subscription

like aws account

1 user --> multiple subscriptions

billing unit --> payment method (ex: credit card, MS corp partnership...)

types:
* free plan: 200$ credit first 30 days + multiple services for free during 12 months
* pay as you go: give credit card and billed as you go
* EA (enterprise agreement): business relationship with MS
* free credits: msdn, startups...

organization --> multiple subscriptions: sales, IT, finance, north america...
   --> reasons: security and separate billing 

# management groups

1 management group --> multiple subscriptions

control subscriptions from a centralized place
    --> by people with different management capabilities
