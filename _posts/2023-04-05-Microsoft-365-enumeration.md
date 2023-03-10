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

**This article is based on TeamFiltration 3.5.0**

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

This method (GetCredentialsType) will not work on each tenant and depends on specifics (managed or federated domains for instance), and might give some false-positives. This is not the purpose of this article, however the current way TeamFiltration (as of v3.5.0) does validate usage of this method is also incorrect.
This is indeed the problem with undocumented APIs, the specification did evolve and thus the same response is not expected. 
This lead to using *--validate-msol* method resulting in failre message: this method is not supported for the target tenant. 
<br />

Note that, when using this method, this is throttled by Microsoft and hence slow if enumerating on a big user list.

When you issue the *--enum-msol* command with a target domain, TeamsFiltration will ask you for an expected email format, so it can then "brute-force" enumeration based on a list of common names (John Smith, Sarah Parker...), pulled from *[statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames)* (typically, the target tenant we use has a user called John Smith, which would be a direct match) or based on a potential list of usernames which you pass as input:

![image](https://user-images.githubusercontent.com/18376283/224321359-6e6a7081-db3e-4f2c-8261-ff77951a3147.png)

![image](https://user-images.githubusercontent.com/18376283/224321851-770b7afc-c220-42e8-91d3-a7d892d0d7d9.png)

**Note:** When you do enumeration, TeamFiltration is building up a database of previous attempts, and will skip the usernames it already tried to enum for this specific domain (database is in the --outpath parameter).

As you notice and as discussed above, the method is not supported by TeamsFiltration on our target tenant.

#### Detection opportunities

One of the problem for blue teams when attackers are using undocumented APIs (understand: APIs used by known applications or websites but not documented for development or customer usage, to the opposite of APIs such as Graph API) is that most of the time, they won't be visible inside your logs or not in the way you'd expect.
Moreover, in this case, there is no login attempt, it is just enumeration. 
The MSOL API is used by Office 365 and a tons of other Microsoft apps for authenticating a user interractively. 
If the authentication happens successfully, sign-in logs will of course appear in Azure AD but the *GetCredentialsType* API will not trigger any.

### Enumeration using Teams (Microsoft Teams APIs)

This enumeration method is the 'core' of the research presented at Defcon, as the author did an extensive analysis of how Teams authentication works and what APIs are called by the Teams client. You can see from the [Defcon presentation](https://www.youtube.com/watch?v=GpZTQHLKelg) that Teams uses quite a lot of APIs, including MSOL, the one we covered just before. However this method is leveraging the Teams search API to search users cross-tenant.
<br />

**Hint for blues:** you can disable the cross-tenant search in Teams Administration pages: https://learn.microsoft.com/en-us/microsoftteams/teams-scoped-directory-search. Do note it has UX impact and is enabled in most tenants. 

This method *does require* a so-called sacrificial O365 account, which is simply an Office 365 user account with at least Teams enabled, in order to do cross-tenant enumeration. 
Of course you could also imagine that an attacker use a single compromised account from your own organization to do the same, hence removing the need for the cross-tenant setting, but potentially raising more flags in terms of detection and, at the end, if you have already an account in the tenant, there are many other ways to enumerate users.

In this test, I used a sacrificial account in a test tenant I own and this time provided a list of usernames (which, again attackers usually easily scrap from many sources) rather than using the brute-force enumeration available in the tool (based on the Github list of common names).

![image](https://user-images.githubusercontent.com/18376283/224323285-6f22ebb3-05c9-4044-8a7f-c3d72defc5ad.png)

You can see that the tool successfully validated 5 accounts from the list input.

#### Detection opportunities

So this one I can detect right? Ehhh...no. 
This is again just enumeration using a valid cross-tenant search feature. 
If we look at _OfficeActivity_ or _SigninLogs_ or yet _NonInterractiveSigninLogs_, there is no trace of such enumeration. Remember, there is no sign-in attempts.
One could argue for the possibility to log this in the [Unified Audit Logs (UAL)](https://learn.microsoft.com/en-us/powershell/module/exchange/search-unifiedauditlog?view=exchange-ps), but imagine the number of logs it could generate, for little value at the end. 

### Enumeration using Logins

The last available method is the simplest and of course most useful one from a detection standpoint. It actually tries to login to see if the user exists or not.

![image](https://user-images.githubusercontent.com/18376283/224324390-5063ce29-aafe-4ed9-911b-2cc8ffdcb7b3.png)

It will trigger sign-in attempts, like you would do by simply browsing to the Office365 portal for instance. It then uses a random password to test if the user exists or not.

#### Detection opportunities

This method is of course the only enumeration method proposed by TeamsFiltration which we can detect as this basically attempt to login with either the provided list of usernames, either by attempting the list of common usernames described above. We are interested therefore in *interractive sign-in logs of Azure AD*.

![image](https://user-images.githubusercontent.com/18376283/224325671-b1e882a9-b7d0-4245-914d-153599ff9d4f.png)
![image](https://user-images.githubusercontent.com/18376283/224325784-f983574d-4cb9-4891-b64a-60c74fff5bae.png)

There is a few interesting details in the logs:
- Location: in this case it is France, but next time it might be US, FireProx will indeed create the AWS endpoints in a random region, so this is not a IOC you could use. However, the fact that these locations might be rare in your company and rolling for each enumeration attempts, might give other detection opportunities.
- Authentication Requirements: You can see if account enumerated has MFA enforced or not, which could higher the related incident severity
- Conditional Access: This was not applied, as the login attempt failed at first factor
- IP Address: This is a public IP from AWS public IP range, could be a completely different one next time, still in AWS range
- Device detail: there is no details about the device and no user-agent mentionned

**Why is there no user-agent and what would be the expected one?**

The user-agent is defined in the configuration file of TeamsFiltration. However, if you indicates an agent which Azure AD cannot match, it will just be empty in the Sign-in logs. Here is the configuration file used for this attempt:

![image](https://user-images.githubusercontent.com/18376283/224327113-5efe54c1-b244-41a7-949c-78f61fcf3518.png)

If we do the same attempt, with a known/valid agent:

![image](https://user-images.githubusercontent.com/18376283/224327400-7fe422ac-c953-4271-95a0-af57eddc27a7.png)
![image](https://user-images.githubusercontent.com/18376283/224329234-9d5a327e-2168-4f24-8881-444c95af7c6f.png)

So this is a bug in the way Azure AD logs handle the user-agent string, as an invalid or empty agent will lead to an empty _DeviceDetails_ field in the logs. 
Do notice as well the changes of IP Address and Location. 
This being said, as you might know, Azure AD is tighly integrated to Defender 365, and two new tables are available on Defender 365 advanced hunting blade, which are probably meant to replace the current __SignInsLogs and NonInterractiveSignInsLogs tables__ at some point:




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

Let's have two real attempts now, where we input a password list from a dictionnary of top 10000 common passwords:

![image](https://user-images.githubusercontent.com/18376283/223654338-133f165e-313d-469b-b0cf-7d2e53400432.png)

...and yes playing with fire, reducing the time between sprays too much and you just locked all accounts.

Let's do the same, but for the purpose of this article to be done before 1 month, just with 5 passwords in the dictionnary and a bit bigger delays:



#### Detection

The two methods will trigger different logs: one is interractive, the other one relies on non-interractive sign-ins, with of course, in both cases, FireProx being used with fresh instances for each attempt, and hence new IPs from AWS public IP ranges.

### Exfiltration

Ok if you have a successful spray or are able to find a valid account and credentials through any other mean, you want to exfiltrate some data potentially. This is where the exfiltration module comes handy. 
It offers several features including:
- Extracting info from the AAD tenant of the user, using Graph API
- Extracting Teams chats, contacts and files through the Teams API
- Extract cookies and authentication tokens from an exfiltrated Teams database
- Extract files from OneDrive or SharePoint
- Extract mails using Outlook API 
- Exfiltrate JWT tokens to abuse SSO mechanism

It also allows to exfiltrate directly from a token as an input or abuse the refresh tokens from the victim's cookies. 

Let's have a deeper look:

![image](https://user-images.githubusercontent.com/18376283/223680440-6ffe4b5d-5f4c-4ef1-98e4-2bef19db2628.png)

#### Detection

OfficeActivity
Sign-in Logs
Non-Interractive Sign-in Logs
Audit Logs 

###  The Backdoor

As a bonus, TeamFiltration proposes a 'backdoor' module. The purpose of this module is basically to interract with the OneDrive API using a custom CLI.
Why would you do that? Because that way you can replace a legitimate file in the user's OneDrive, by the same file with a malicious macro or a HTA file embedded. 
You can look at dates to see how often the user is using some files, to grow chances of the payload to be executed.
Another nice trick mentionned in the Defcon talk is that the author realized the Desktop folder is often sync'd to OneDrive by default. The desktop folder is often a set of shortcuts (LNK files). You can therefore simply replace one of the shortcut to point to a malicious payload or an executable you'd like to launch with user's privileges. 

#### Detection 

None?


## KQL Queries - Detecting / hunting

## Protecting 

- MFA
- CA policies
- Spray protection
- Password complexity 
- AAD IP\
- ...
