---
layout: posts
Excerpt:  "Azure AD Applications - Making sense of available logs"
title:  "Azure AD Applications - Making sense of available logs"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

**Azure AD applications, or application objects, can lead to several attack vectors, covered in multiple documentation pages and blogs already. It can be confusing for analysts to navigate through Azure AD terminology and logs and quickly come to a conclusion on a hunt or during incident response. In this post, I am going over relevant logs, what they mean, normal and unexpected behiaviors.**


I have broken this article into three parts: 
- Setting the scene: a brief summary of the terminology around Azure AD applications  
- Understanding normal: what Azure AD application events are logged and where 
- Detecting anomalies: how threats can be identified in available logs 

## Azure AD Applications

Azure AD applications are covered in numerous blogs or in the official Microsoft documentation pages, but let's start by going over the concept of an Azure AD application and what it means in practice. <br /><br/>

The term *application* can be a bit misleading in the context of Azure AD. <br /> 
You can see an Azure AD application as being an identity (which we will refer to as a **Service Principal**) and some configuration settings (which we will refer to as an **Applicatiob Oject**) for an application (web, mobile...) in an Azure AD tenant. <br />
<br /> 
**Note:** An application can be available in multiple tenants, but more on that later. 

When an application is added to Azure Active Directory, it could be for several purposes:

- Authenticate and authorize (RBAC) users to the target application
- Single-Sign On SSO in the company tenant 
- Taking actions on-behalf of users or other applications
- Leverage OAuth to authorize the specific application to access Microsoft 365 or Azure AD APIs/resources
- A modernization proxy - Publish an on-premises application to the internet
- ...

### The application settings, as known as: the application object



### The application identity, as known as: the service principal


## Detecting Azure AD Application threats using Azure AD logs
