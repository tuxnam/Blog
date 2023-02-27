---
layout: posts
Excerpt:  "Detecting Microsoft 365 enumeration - TeamFiltration in the spotlight"
title:  "Detecting Microsoft 365 enumeration - TeamFiltration in the spotlight"
last_modified_at: 20@#-01-28T15:59:57-04:00
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
As the framework is open-source, attackers could easily change or remove the user of Fireprox to adapt to their own infrastructures/needs. 

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

#### How TeamsFiltration use it

If we run the MSOL enumeration module with TeamFiltration, and a proxy in place, such as BURP, here is what we get:

The following paramenters are hardcoded (at time of writing) in the framework code for this method:
```
isOtherIdpSupported = true,
checkPhones = false,
isRemoteNGCSupported = true,
isCookieBannerShown = false,
isFidoSupported = true,
originalRequest = "",
country = "US",
forceotclogin = false,
isExternalFederationDisallowed = false,
isRemoteConnectSupported = false,
federationFlags = 0,
isSignup = false,
flowToken = "",
isAccessPassSupported = true
```

#### How noisy is this?









3. Using Teams
4. Using Office 365 Logins

### Spraying

### Exfiltration

## Our Microsoft 365 test environment

## The attacker scenarios 

## Diving into the logs

## Yara and KQL detection rules / hunting
