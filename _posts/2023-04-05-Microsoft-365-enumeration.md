---
layout: posts
Excerpt:  "Detecting Microsoft 365 enumeration - TeamFiltration in the spotlight"
title:  "Detecting Microsoft 365 enumeration - TeamFiltration in the spotlight"
last_modified_at: 20@#-03-28T15:59:57-04:00
---

*TeamFiltration is self-defined as a cross-platform framework for enumerating, spraying, exfiltrating, and backdooring O365 AAD accounts. <br />
In this article, we will look at its capabilities and how we can potentially detect related events and activities.*


## Introducing TeamFiltration

[TeamFiltration](https://github.com/Flangvik/TeamFiltration) is a framework developed by @flangvik from TrustedSec which allows to perform reconnaissance and gain initial access to Microsoft 365 and Azure AD tenants in order to potentially exfiltrate data or open the door to other post-exploitation activities. 

## Purpose of this article

Enumeration, spraying and brute-forcing are common attack techniques for initial access ([TA000! - Initial Access](https://attack.mitre.org/tactics/TA0001/)), specifically on Cloud workloads. 
While there are loads of interesting articles on post-exploitation frameworks, "post-exploitations", by definition, means there was an initial access. Detecting initial access attacks and analyzing relevant logs in Azure AD is what we strive to do in this article. 
TeamsFiltration is of course not the only framework available for enumerating or brute-forcing Azure Active Directory (e.g.: PowerZure, Microburst, MSOLSPray and many more) but it allows to chain multiple techniques, some of them being interesting from a detection perspective.

We will divide this article in several sections, starting with disseting each one of the technique offered by TeamsFiltration and looking at the corresponding detection capabilities. 

TeamFiltration is written in C# and can either be used from a standalone build version or by compiling sources from the corresponding GitHub repository.
I encourage you to watch the presentation of the framework [@Def Con 30 - Taking a Dump in the Cloud](https://www.youtube.com/watch?v=GpZTQHLKelg) for an onveriew of the framework, its foundations and a demo by the author. 

TeamFiltration has 3 main modules: enumeration, spraying and exfiltration. Let's dive into the inner-workings...

## A note on Fireprox 

TeamFiltration does not trigger enumerations or spraying directly from the client IP where you execute it, it is a bit more clever and leverages a proxy utility called Fireprox. Fireprox generates pass-through proxies that rotate the source IP address with every request, leveraging the AWS API gateway behind the scenes.
More information can be found [here](https://kalilinuxtutorials.com/fireprox-aws-api-gateway-creating-http/).
When you execute TeamFiltration, it will actually first create a temporary AWS API gateway and corresponding endpoints.
Example for *enum* and *--validate-msol* technique:

```
https://{API-ID}.execute-api.{YOUR-AWS-REGION}.amazonaws.com/fireprox/common/GetCredentialType
```

When called, this endpoint will simply forward requests (as a proxy) to https://login.microsoftonline.com/common/GetCredentialsType

This allows users of TeamFiltration in this case to "hide" their real IPs when scanning and use IPs from AWS public IP range, which will keep on being rotated and hence avoiding a simple IP block. 

From a detection standpoint, however, this also means that enumeration and spraying from AWS public IP ranges will be a signature of TeamFiltration.

**Note:** the framework might evolve in the future and bring similar capabilities on Azure or GCP or even bring VPS capabilities in, but in the current state, AWS public ip ranges is a good IOC to build on.
As the framework is open-source, attackers could change or remove the user of Fireprox to adapt to their own infrastructures/needs. 

## Enumeration

TeamsFiltration enumeration has three parameters which allow to enter a domain for the target organization (example *contoso.net*), adding an optional list of usernames (common usernames or for instance a list gathered using OSINT, Google, LinkedIn, a company portal or other sources) or for instance using the *dehashed* (a database of compromised assets such as email accounts). The enumeration module offers three *enumeration types*: MSOL, Teams or regular logins attempts. 

### Enumeration using MSOL (Microsoft Online module)

#### The method

This enumeration is using a known technique based on the *login.microsoftonline.com* API, which is simply the API used by all users to login to Office 365. If you authenticate to https://www.office.com/, you will be able to see the OAUTH flow leveraging this specific API. 
The exact method used by this enumeration technique is *GetCredentialType* which gives login information, including Desktop SSO information.
What TeamsFiltration will do, when you use this method, is generating a random username (in the domain of the target organization), and validating it against *GetCredentialType*. 
This technique has been covered multiple times in the past. This API method expects several parameters, we can see this by simply fuzzing the endpoint with Postman and a valid username to start with:

![image](https://user-images.githubusercontent.com/18376283/221423725-21fc7136-e52d-4d6b-9ad3-1a2a239a6fa7.png)

If we compare with an account which does not normally exists on this domain:

![image](https://user-images.githubusercontent.com/18376283/221423809-df597f7a-2c4e-4faf-8352-805987a023b6.png)

The notable difference is on "IfExistsResult" where it will respectively be 0 or 1 depending if the user exist in the tenant or not. 
It directly shows you how easy it is to enumerate accounts using this method.

This method (GetCredentialsType) will not work on each tenant and depends on specifics (managed or federated domains for instance), and might give some false-positives. 
However, the current way TeamFiltration (as of v3.5.0) does validate usage of this method is also a bit incorrect and lead to the attempt telling most of the times that this method is not supported for the target tenant. This is one of the problem (compared to the advantages it brings from a detection standpoint) to use undocumented APIs which can change often and lead to inconsistencies. <br />
I will dive a bit more into *GetCredentialType* and how to use it for enumeration in another article, for now we will focus on TeamFiltration's capabilities.

Note that, when working for your tenant, this method is throttled by Microsoft and hence slow if enumerating on a big user list.

#### Results on our test tenant

When you issue the *--enum-msol* command with a target domain, TeamsFiltration will ask you for an expected email format, so it can then "brute-force" enumeration based on a list of common names (John Smith, Sarah Parker...), pulled from *[statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames)* or based on a potential list of usernames which you pass as input:

![image](https://user-images.githubusercontent.com/18376283/222366500-37bdd627-06d5-4ca6-af57-10a77f228a21.png)

![image](https://user-images.githubusercontent.com/18376283/222367191-ffd4aff3-50e3-4221-b459-b6c289d260ea.png)

**Note:** When you do enumeration, TeamFiltration is building up a database of previous attempts, and will skip the usernames it already tried to enum for this specific domain (database is in the --outpath parameter).

As you note, the method is not supported by TeamsFiltration on our target tenant.

#### Detection opportunities

One of the problem for blue teams of attackers using undocumented APIs (read: APIs used by known applications or websites but not documented for development or customer use to the opposite of on-purpose APIs such as Graph API) is that most of the time, they won't be visible inside your tenant.
The MSOL API is used by Office 365 and a tons of other Microsoft apps for authenticating a user interractively. If the authentication happens successfully, sign-in logs will of course appear in Azure AD but the *GetCredentialsType* API method does not require any authentication attempt and will be blind from a Azure AD or Office 365 UAL logs standpoint. 

### Enumeration using Teams (Microsoft Teams APIs)

This enumeration method is the 'core' of the research presented at Defcon, as the author did an extensive analysis of how Teams authentication works and what APIs are called by the Teams client. You can see from the presentation that Teams uses quite a lot of APIs, including MSOL, the one we covered just before. However the method used by TeamFiltration here is simply to use the Teams search API to search users cross-tenant.

**Hint for blues:** you can disable the cross-tenant search in Teams Administration pages: https://learn.microsoft.com/en-us/microsoftteams/teams-scoped-directory-search.

This method *does require* a so-called sacrificial O365 account, which is simply an Office 365 user account with at least Teams enabled, in order to do cross-tenant enumeration. Of course you could also imagine that an attacker could use a single compromised account from your own organization to do the same, hence removing the need for the cross-tenant setting, but potentially raisong more flags in terms of detection and if you have already an account in the tenant, there are many ways to enumerate other users.

In our case, I used a sacrificial account in a test tenant I own and this time provided a list of usernames (which, again you can gather from many sources) rather than using the brute-force enumeration available in the tool.

![image](https://user-images.githubusercontent.com/18376283/222748485-0316e9e0-e639-46f7-b4b8-bebfe63debde.png)

You can see a few valid accounts have been marked as such.

#### Detection opportunities

So this one I can detect right? Ehhh...no. This is again just enumeration using a valid cross-tenant search feature. If we look at _OfficeActivity_ or _SigninLogs_ or yet _NonInterractiveSigninLogs_, there is no trace of such enumeration:

![image](https://user-images.githubusercontent.com/18376283/222757901-ff5c9feb-f2b3-4e0d-ba38-faa63968f344.png)


### Enumeration using Logins

This method is the simplest and of course most useful one from a detection standpoint. It actually tries to login to see if the user exists or not.

![image](https://user-images.githubusercontent.com/18376283/222759774-bd11bc44-3aed-409d-930b-b8ab741ca964.png)

This just basically uses https://login.microsoftonline.com like you would do by simply browsing to the Office365 portal for instance. It then uses a password to test if the user exists or not.

#### Detection opportunities






3. Using Teams
4. Using Office 365 Logins

### Spraying

### Exfiltration

## Our Microsoft 365 test environment

## The attacker scenarios 

## Diving into the logs

## Yara and KQL detection rules / hunting
