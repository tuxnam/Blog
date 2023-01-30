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

Think of application settings as a configuration file/database for your application. It is called the **applicaton object** in Azure AD and allows to configure several aspects of your application:
- A name, description, logo
- Some redirect URIs which can be used during authentication flows for instance
- Secrets and certificates 
- Application roles
- Exposed APIs
- ...

The application objects can be created in multiple ways, by an application administrator in a specific tenant: Azure AD GUI, PowerShell, through an IDE, from the Azure Application gallery.

The application objects for a specific tenant will appear in the **Application Registrations** blade of Azure AD:

![image](https://user-images.githubusercontent.com/18376283/215504979-24a10457-4303-44a0-a5fe-17d9529b1f5d.png)


**Step 1 - Let's create an application**

In the Azure AD application registration blade, let's register an application:

![image](https://user-images.githubusercontent.com/18376283/215505119-f0677c29-c895-4b5b-89b8-9b79a988340f.png)

We will make it single-tenant for now, and after completion, you can observe an entry has been added to the registrations as well as properties (in green) and settings (in orange) for that new application:

![image](https://user-images.githubusercontent.com/18376283/215505786-ebd9e43b-b0c8-40fb-9768-5c137dff8f77.png)

The properties are the first part of artifacts I want to describe, as they are important from an analyst perspective and will reflect in the available logs:

| Property | Description |
| ----------- | ----------- |
| Display Name | Title |
| Application (client) ID | Title |
| Object ID | Text |
| Directory (tenant) ID | Text |
| Supported account types | Text |

A few settings are also interesting to look at:

- Owners
- Authentication
- Certificate and Secrets
- API permissions 
- Roles and administrators

Our application now has properties and settings in the target tenant. 
Any object in an Azure AD tenant however requires permissions to perform actions, how does this work with an application? 
This is where the identity part kicks in, with the infamous service principals.


### The application identity, as known as: the service principal


## Detecting Azure AD Application threats using Azure AD logs
