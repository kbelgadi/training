# Factors that affect cost 

on-premises env: 
* most costs fixed, 
* even monthly variable cost is known based on contracts with providers
--> no flexibility to add/remove services as you require

one of the benefits migrating to cloud: 
* saving money, 
* flexibility in how you pay for services: pay for period (ex: month), pay for perfs, pay for amount of resources (ex: cpu)
--> certainty over what could the cost be at the end of the month
    pay for services that charge by consumption

Factors that affect cost:

* time: services that charge by time spent using the service
ex: VM cost per hour, but does not include the storage (hard disk, OS disk, temp storage, etc) -- you pay separately for that
Additional thngs to pay for: ip addresses, LB, etc

* consumption (storage, cpu, bandwidth): in addition to time, you could pay for the amount of consumed resource
ex: azure function --> you pay by usage (compute consumption)
any data livig azure for the internet will cost you -- or inter-regions transfer -- limited free outbound 
(no limit for inbound transfer to azure)
services used but not consumed will have a cost
transaction cost: read, write, list, etc could be charged differently

* service tier
(ex: basic, std, premium)

* computing power (vcpu, ram, cpu type)
ex: cosmos db, sql db --> you pay by type of cpu, ram, etc

* software licences 
you can save money on Windows licenses if included in enterprise agreement with MS, sql server, etc

pay attention to the several options for each service:
ex: linux --> 12 options for the distros
sql server --> dev, enterprise, etc

a cheaper way exists, but without SLA (garanteed amount of service), slower latency

pay for fixed monthly --> certainty in the cost, but you may pay more than what you consume really
pay by consumption --> cheaper, but no cost predictability

# Cost calculator

Simulator of bill in  azure portal
Simulation of cost based on multiple parameters: resource, consumption time, tiers, location, etc

# TCO (Total Cost of Ownership) calculator 

simulator that do comparison between running your solution in azure and all the cost if you try to run them yourself on-premises

ex of server:
same cost if we consider only the cost of buyin and running the server
the difference is obvious if you include the cost of running/maintaining the server: 
    --> infra cost (network equipment, internet plan, secured DC), labor (install OS, softwares, patches), electricity, cooling

result of simulator:
cost of running a number of servers during 5 years
the difference between azure and on-premises

# Azure cost management

how much you are spending within azure (combien vous dépensez sur azure)

a tool that: 
* displays past invoice: historic of invoices, report daily, weekly, monthly basis
* helps analyze spending within azure
* set a budget. Ex: no more than 100$ --> a warning is sent by email when approachin this limit
* you can compare the spending this month / last month / the month before
* forecast the spending for the current month

to properly track spending 
    --> tag resources properly: a code added as a tag for the resource to be charged to the right client's account

# Resource tags

Tags provide metadata that can be added to Azure resources to organize related resources and help with billing and support issues

can be added during resource creation or after the resource has been created

in cost management console: group cost by tags