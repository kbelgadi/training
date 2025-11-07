# Identity and MS Entra ID (formerly: Azure Active Directory)

bad idea: roll your own code to handle security

hack occured on folowing breaches:
* password stored in plain text
* using outdated md5
* storing salt with data
* weak password complexity
* poor password chnge policy

MS Active Directory: access to laptop in corp env
Cloud extension of AD: MS Entra ID (called in the past: Azure AD)

Differences
* AD: 
    run on a server, controls different ypes of objetcs
    uses protocols not designed to work over the internet: LDAP, Kerberos
* MS Entra ID: 
    cloud focus
    use SAML and OAuth protocols

instead of coding your own security --> use entra id to handle authent autoriz

client -------- web server
   \              /
    \            /
        identity 
        provider

1. auth on id provider with login/passwd
2. receive a token
3. submit the token to the web server
4. web server verifies the token is valid because it is signed by the id provider

# Benefits of MS Entra ID

on entra id:
use corp userId/Password
users can create their own user id / password in the cloud
use of social media logins: facebook, google, ms account (B2C solution)

reduce dev time
easier support

MS provide its own support to entra id: no expected crashes, etc

get unique id and key --> use it in the cde to pass off the login to entra id (instead of login/password screens)

self service support: to reset a lost password, 
2FA, etc

in addition to login/password, other features
* check with AI id login pattern matches existing threats
* different login requirements, based on usage from crp network or from personal network in case of remote working
* audit feature every year: check if peaople still need acess to things

as administrator:
central support of all users and permssions
flag some users to use 2FA

SSO: sync with corporate AD
corp users creds recognized in the cloud
usage of the same cred to login to on-prem apps and on azure apps
password change is reflected in the other location
if permissions revoked, revoked in the other locations

entra id intergrates with other azure services
ex: use of entra id to assign perms to storage accounts, virtual machines, etc

# Authent VS Authorization

Auth: users proving who they are 
ex: 
userid/password
MFA: contac on a secondary device to make sure you are in control of your phone or email-account
thumbprint identification, face recognition, pin code

Authorization: we assume you are authenticated, but what permissions do you have?
ex: via RBAC

Recommandation: give peaople only the level of authorizations they are entitled to
ex: run reports, but without being authorized to modify data

# Azure AD conditional access

entra id has a subscription model: free tier, premium one tier (P1), premium two tier (P2)
conditional access is available from P1

conditional access: not every attemt to login to your system is equally safe
--> rank certain attributes of login as normal and expected VS things suspiscious and unexpected
--> if supiscious: additional steps before to logging in

ex:
safest attemps to log in:
login from inside corp building with the same device as everyday: normal 
suspiscious:
login from foreign country (not where corp offices are located): unexpcted
somemoby not logged in for months, suddenly try to login: unexpected
logging in from device never used before: unexpected
--> provide an extra step, or block access

# MFA, 2FA

MFA: requirement of 2 or more pieces of evidence (factors) in oreder to login

user id / password: one factor
if usage in web site, and these went hacked 
    --> account compromized 
    --> places where same password used, your account is compromized too

Factors categories:
* something you know
* something tou have
* something you are
several types of identities for each

you know:
user name: not a factor --> it is a public information (ex: email address, nickname, etc) --> not enough secured
password: not exactly a factor --> something you know, but should be kept private

you have:
mobile phone: usage of code send by sms by an app to authenticat --> sim card cam be spoofed 
some modern systems moved to authentication apps (MS, Google, etc) --> mode secured than an sms

fob with number that changes avery N sec --> similar to authen apps

you are: 
finger print, face scan 

combination of factors is harder to hack (ex: fingerprint + text msg from auth app or password)

MS Entra id:
admin accounts have MFA configured (free feature)

Factors supported by MS entra id:
MS authen app, windows hello (pin to login into windows account, because tight to the hardware, encrypted locally)
FIDO2 (fob)
SMS
...

# Passwordless

alternative to mfa

passwords: convenient but low security
password + 2FA: unconvenient but high security

passwordless authen: convenient and high security
ex: 
using  gesture to sign (draw to login to mobile phone)
pin code, biometric recognition, etc

all captured data kept within the device, not shared with MS

Windows hello: bluetooth link with phone --> get up away from desk => device shutdown

# RBAC

token-based control is migrating to rbac control

hundreds of built-in roles in entra id
or create custom roles

distinguish between: admin of entra (entra admin) VS admin of services within azure account 

role: a set of permissions

add user to a role to authorize him to do certain operations
remove user from role to deny permissions

principle of least permissions: 
ex: users shouldn't have full admin level on azure account
don't give too high permissions to persons that do not expect, especially in a one-time exception

possible to create custom role that is comprised of built-in roles

basic roles in entra:
* reader: readonly access. Ex: access to storge account, but you can't chage anything
* contributor: full access to resource, but can't share the authrozations with other peaple
* owner: full access to a particular resource or to all resources, allows to assign permissions to other

# Zero-trust model of security

back 20 to 30 years back: 
--> IT security focused on protectig the border, the boudary between comp and internet
--> everyting inside the corp net is assumed safe

pb: if a haker get inside the comp net don't want everything becomes available to him

zero-trust model: 
you can't trust any connection or request regardless of where it comes from
you force every request to incude a proof of who's making it

zero-trust principles:
* verify explicitly: instead of using the same token all the day, do more intence verification
* least privilege access: identify who has access to what, don't give full admin access to everyonme whe requests it
* assume breach: consider the risk of an app in a server that gathers informations --> design comm, data storage accordingly

use every available methods to identify identtities: user/password, ip address, geo loc, etc

JIT (Just In-Time) access: elevation of privileges for limited period (ex: few minutes) to do the task
JEA (Just Enough Access)

Secuity even inside the network:
* encryption
* segmentation: access to the service for only app and users who need 
* threat detection: monitor traffic and detect suspiscious behavior

verify identity (for access to apps, etc)
devices in compliance of comp policy (uptodate sys, browsers, etc)
in-app permissions: monitor who's logging in, what resources accessed. Put logs in accessible place
secured data: encrypted, restrict access, understand what you have, etc
secured infra: monitor to detect attacks
secured net: encrypte comm

# Defense in depth

the more defense, the more secure

mix and match of sec measures in place:
* data: virt net endpoints
* app: api management
* compute: limit remote desktop access, windows update, use bastion server
* net: nsg (network security groups: deny for all conx by default except for specific approvals), subnets, deny by default
* perimeter: ddos, firwalls
* identity & access: entra id
* physical: doors locks and key cards

defense in depth: having secure sol all accross apps and apps layers

You can restrict traffic to multiple virtual networks in multiple subscriptions with a single Azure firewall. 
Azure Firewall is a managed, cloud-based network security service that protects your Azure Virtual Network resources. 
It’s a fully stateful firewall as a service with built-in high availability and unrestricted cloud scalability

# MS defender for cloud

Security posture management and threat management

various products to enhance security of your services

in the right of the MS defender service console, the services that you can protect:
servers, app service instances, azure sql db, etc
cost: per server per month, or per number of TX

30 days free trial available for defender for cloud

can give:
analysis of sec posture, recommendations
sec score
regulatory compliancy (HIPAA, PCI, etc)
inventory of resources

