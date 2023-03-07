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

![image](https://user-images.githubusercontent.com/18376283/223357820-8c14ca81-8e6f-4f72-89ce-0d2ddcfbff32.png)

It will trigger sign-ins like you would do by simply browsing to the Office365 portal for instance. It then uses a password to test if the user exists or not.

#### Detection opportunities

This method is of course the only enumeration method proposed by TeamsFiltration which we can detect as this basically attempt to login with either the provided list of usernames, either by attempting the list of common usernames described above.

If we look at regular user sign-ins we can clearly see the attempts, as well as specifics from TeamsFiltration:

![image](https://user-images.githubusercontent.com/18376283/223469262-b986a087-0af3-4521-a928-eef0ce2e001b.png)

_Wait, what with the different Application IDs and Application Display Names? (in green)_

Good catch! In fact, TeamsFiltration does not always use the same APIs in this method, and rotate between several (hardcoded) APIs, but more specifically it enforced specific Application IDs which are corresponding to specific applications in any Azure AD tenant.
The full list of applications and application IDs of Microsoft first-party applications can be found hbere: https://learn.microsoft.com/en-us/troubleshoot/azure/active-directory/verify-first-party-apps-sign-in#application-ids-of-commonly-used-microsoft-applications. 

**APIs used:**

https://graph.windows.net<br/>
https://api.spaces.skype.com<br/>
https://outlook.office365.com<br/>
https://management.core.windows.net<br/>
https://graph.microsoft.com<br/>

With the following Application ID, "1fec8e78-bce4-4aaf-ab1b-5451cc387264", which if you look in Azure AD, like confirmed by the above list, corresponds to Microsoft Teams (it will be the same GUID for all Azure AD tenant):

![image](https://user-images.githubusercontent.com/18376283/223362726-340e4832-787f-46ad-8946-548798d8cf52.png)

https://clients.config.office.net<br/>
https://onedrive.live.com<br/>
https://wns.windows.com<br/>

With the following Application ID, "d3590ed6-52b3-4102-aeff-aad2292ab01c", which corresponds to Microsoft Office. 

https://api.office.net<br/>
https://onedrive.live.com<br/>
https://clients.config.office.net<br/>
https://wns.windows.com<br/>

With the following Application ID, "ab9b8c07-8f02-4f72-87fa-80105867a763", which corresponds to OneDrive SyncEngine.

Of course the tool will evolve, and could rotate between all application IDs available [here](https://learn.microsoft.com/en-us/troubleshoot/azure/active-directory/verify-first-party-apps-sign-in), so detections should not be limited to the current list.  

__IPs used: AWS APIs__

As we discussed earlier, TeamsFiltration will leverage Fireprox by default and hence the origin IPs will all be in the AWS Public IP range:

![image](https://user-images.githubusercontent.com/18376283/223469741-d20742ea-f175-447a-b36d-06b38091e0fc.png)

Of course, AWS IP ranges are huge and assigned based on service and region. 

__User Agent__

One other thing to note is the user-agent string, which in this case in "Chrome 80.0.3987". This user-agent is in fact hardcoded in the TeamFiltration configuration file and can thus be changed by any user of the tool. If no agent is specified, the default String will be: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Teams/1.3.00.30866 Chrome/80.0.3987.165 Electron/8.5.1 Safari/537.36", which corresponds to Chrome browser: see [here](http://www.browser-info.net/useragent?q=Mozilla%2F5.0+%28Windows+NT+10.0%3B+Win64%3B+x64%29+AppleWebKit%2F537.36+%28KHTML%2C+like+Gecko%29+Teams%2F1.3.00.30866+Chrome%2F80.0.3987.165+Electron%2F8.5.1+Safari%2F537.36).

**Let's do this one more time**

If we run the same enum command again, clearing the database first (TeamsFiltration is re-using previous attempts of enumeration or spraying for efficiency and storing it all in its embedded database) but also changing the user-agent to a non-existing one:

![image](https://user-images.githubusercontent.com/18376283/223472620-ede86933-1337-4c1e-a277-1e6d373e4a5b.png)

If we look at the logs, we see the device details have changed, and are not available anymore, but also the AWS IP range is different, as it used new FireProx endpoints generated on-the-fly and the applications used are different as well with for instance OneDrive Sync Engine being used:

![image](https://user-images.githubusercontent.com/18376283/223474014-6101ff53-f933-4013-9445-04af0a7f95c3.png)

**Results of the authentication attempts**

For all these attempts, unless of course the password generated by TeamsFiltration (which is one of these) is correct, it will generate the following status code: 50126 - Invalid username or password. Notice as well the location of the AWS IPs changing on the second run from FR to US:

![image](https://user-images.githubusercontent.com/18376283/223475099-96208304-9bcd-4e2c-b274-d1dbe67265ce.png)

**Wait a minute..."

If error code is invalid username or password, how does TeamsFIltration knows these are valid accounts? 
Well because in fact if the account does not exist at all, the response issued by the above APIs will be different.Let's try with https://outlook.office365.com:  

![image](https://user-images.githubusercontent.com/18376283/223477810-9e19eb71-5dfd-4bed-8ee0-d63da36555c8.png)

Notice, it uses the *GetCredentialType* we discussed at the beginning? now for a valid account:

![image](https://user-images.githubusercontent.com/18376283/223477121-d4e6cdcc-80b1-411f-b540-57e0ccae8931.png)

Wait a minute, it did not even use GetCredentialsType here? Good catch! But is just because my browser had some cached credentials for this one, if I clear the cache and start again:

![image](https://user-images.githubusercontent.com/18376283/223478668-b917501b-54b0-43e7-9462-9544b74ce255.png)


#### A note on the embedded database

TeamsFiltration has an embedded database, which allows to store all enum, spraying or exfiltration attempts locally. If we look into it, we can see the results of our attempts which can be further used in the spraying module for instance!

![image](https://user-images.githubusercontent.com/18376283/223480360-3bef6533-1888-4609-b7ec-6b80af0f5a07.png)


Let's now dive into the tool and analyse the spraying part!


### Spraying

We had a good overview of the enumeration capabilities but enumeration on its own is pointless, the goal is to generally try to gain access (unless you just want to sell login data) and spraying is an efficient way to do so. 

If you want to know more about password spraying and how to prevent it on Azure AD, have a look [here](https://www.microsoft.com/en-us/security/blog/2020/04/23/protecting-organization-password-spray-attacks/).

#### Available spraying methods

TeamFIltration proposes two main methods and several options for password spraying:
- Default method: using standard login method (regular user sign-ins)
- Method 2: _AAD SSO_: this is a method discovered by SecureWorks, described [here](https://www.secureworks.com/research/undetected-azure-active-directory-brute-force-attacks), based on Azure AD single-sign on, and exploiting the target URI: https://autologon.microsoftazuread-sso.com//winauth/trust/2005/windowstransport.

Let's explore these two methods.

![image](https://user-images.githubusercontent.com/18376283/223545437-13319b3f-8ba3-4f0d-88a1-19e17b2a65ba.png)

![image](https://user-images.githubusercontent.com/18376283/223545081-ceb08fb9-2045-46b7-8356-cd2470a47d44.png)

There are several options, we will not explore all of them here, but here is a short summary:

1. The golden rule when you spray is low-and slow, and the default TeamFiltration config is to sleep minimum 60 minutes between each round of attempts and maximum 100. Understand by that, that TeamsFiltration will attempt the list of enumerated usernames with a first password, then sleep before moving to the next password attempt. I used _--force_ here for obvious reasons, to skip the time thresholds, but this of course is needed normally to avoid locking accounts. The "recurring" spraying pattern, albeit changeable, also means a pattern for detection but TeamsFiltration nicely do the next attempts at a random time between the minimum and maximum defined. 
2. You can define a time window when spraying should occur, for instance during business hours, to trigger less anomalies
3. You can input a list of passwords, from a dictionnary, which is recommended. If you don't TeamFiltration generates a list of common passwords from a combination of months, years and generic words. There are several other options like top 20 common passwords for instance. 
4. You can input a list of 'combos', meaning a list of username:password 
5. Next to time windows for spraying, you can define the default delay between attempts
6. There is an integration with pushover to get notifications about successful sprays, so an attacker could have big time windows and delays between attempts, running for months

#### Detection

The two methods will trigger different logs: one is interractive, the other one relies on non-interractive sign-ins, with of course, in both cases, FireProx being used with fresh instances for each attempt, and hence new IPs from AWS public IP ranges.

#### The various methods available for spraying in TeamsFiltration



### Exfiltration

## Our Microsoft 365 test environment

## The attacker scenarios 

## Diving into the logs

## Yara and KQL detection rules / hunting
