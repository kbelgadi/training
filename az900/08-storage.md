# Overview

ToC:
types of storage:
* unmagaed storage: GPv2 (General Purpose Storage)
* managed storage: disque storage - virtual machine virtual disk
file storage
access tiers
storage is IaaS

Azure storage (GPv2)
standard storage
4 types of data: 
* container (blob storage), 
* file
* queue, table (not covered in the exam)
total storage account of all types of data: 5 PiB (5 MiB of GiB)
~ 2 cents / GiB
GPv2 not recommended for high-demand workload (web server, DB)

## GPv2 storage account --> possibility to transform to a a datalake
good for big data
use different protocols to store data
can go beyond 5 PiB limit

## Premimum tiers 
for performance
two types of blobs: block blobs, page blobs
not for queues or tables --> not a multipurpose storage
file storage
--> SSD disk premium high perf ssd
3x OPS compared to standard SSD => lower latency 
premium ssd
premium ssd v2
ultra disk (needs specific material in specific region)

# Container storage (blob storage)

blob (binary large object) for unstructured data: txt, pdf, json, zip... 
in form of a box where you throw whatever you want in there
vailable to anyone in the internet: not possible to set public/private access
 
in a gpv2 account --> create multiple containers
a blob can be part of only one container
concept of folder, but not in term hierarchical concept as in linux/windows filesystems

equivalenet to AWS S3

possible to create multiple storage accounts in any region
good design to store data close to customers
ex: application and its contaner in the same region

region affects prices

## redundancy
by defult: azure keeps 3 copies of your data

locally redundant storage: 3 copies in the same DC
zone redundant storage: 3 copies in the same region

11 zeros of redundancy: 1 file lost in a million years

global redundancy
6 copies = 3 by default locally + 3 others in other regions 
data is not leaving the region

## Access tiers
depending on access tiers case: storing data is cheaper than accessing it... ou l'inverse

* hot: default when creating a storage account
good balance between cost of storing/retrieving data
* cool: cheaper
you can set it as the default
little cheaper to store, more expensive to r/w
ex: for log files
* cold: even cheaper to store
little bit more expensive to r/w
* archive: basically offline
cannot get immediate access, need several hours to get file out storage
cheapest to store
more expensive to retrieve
ex: legal files

## Failover

if azure disks get lst, two additional copies of files exist
disk are recreating in behind the scene


# AZ Copy

copy between 2 storage accounts in different subscripts or in the same one

how to copy data between blob containers?

worst solution: source storage account --> containers --> download locally
    cons: duration and bandwidth cost

better solution: download inside of azure network

CLI: azcopy --> to download/upload local/azure or between azure account into azue network

source storage account --> containers --> container --> file --> select: SAS (Shared Access Signature) --> check permissions: "read"
    --> copy source URL
dest storage account --> containers --> container --> select: SAS (Shared Access Signature) --> check permissions: "write" + all others perms
    --> copy dest url
in local machine --> shell --> use azcopy:
    azcopy {source url} {dest url}

# Azure file storage

slighty different than cont storage

true hierchical structure with folders

possibility to mount the storage to a serveur and refer it by a drive letter ("S:" for windows)

supports SMB (windows, linux) or NFS (linux only)

in corp env: storage server with folders/files

why need of file storage?

replacement of on-premise file storage or additional capacity storage

for lift and shift migration: no changes needed in operations on-prem VS cloud

on-prem: do your backup, add physical disks to extend storage capacity, etc

azure file storage: redunancy option, datarecovery, failover, easier operations

## azure file sync

hybrid option (cloud tiering): most popular files locally (20 GiB), the others on the cloud (20 TiB)

files accessible all around the world

cloud backup of local files

# Azure migrate

to move to the cloud: ambitious without too much disrutpion

=> azure migrate: guided experience

based on your own env, what is the best practices for moving to the cloud?

azure migrate tool:
discovery tool: 
to look to physical/virtual existing servers on-prem or in other cloud
what you gona need in azure: sql server, iis web apps, etc
--> make a plan: left and shift, upgrade, etc

# Azure data box (suite of products)

before to move to the cloud: look to the quantity of data you have to move

it might be umpractical to uplod data depending on the quantity --> it may take too much time

Azure Data products to use for that:
Data box: contains 100 TiB
Data box disk: 8 TiB
Data box heavy: 1 PiB

your env data --> data box 
    --> ship it by courrier to azure 
    --> from azure upload to your account (few days)

data is encrypted into the data box
after upload to account, the data box is cleaned before to be used by another customer







