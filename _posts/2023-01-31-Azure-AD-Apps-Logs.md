---
layout: posts
Excerpt:  "Azure AD Applications - Making sense of available logs"
title:  "Azure AD Applications - Making sense of available logs"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

**Azure AD applications, or application objects, can lead to several attack vectors. While the topic has been covered in multiple documentation pages and blogs already (see references), it can be confusing for analysts to navigate through Azure AD terminology and logs and quickly come to a conclusion on a hunt or during incident response. In this post, I am going over relevant artifacts, what they mean, where to find them as well as understanding application behiaviors from logs.**

I have broken this article into three parts: 
- Setting the scene: terminology, artifacts and log sources 
- Understanding normal: a look at logs from regular Azure AD application events
- Detecting anomalies: how threats can be identified in the same logs 

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

Let's create an application. In the Azure AD application registration blade, we can register an application:

![image](https://user-images.githubusercontent.com/18376283/215505119-f0677c29-c895-4b5b-89b8-9b79a988340f.png)

We will make it single-tenant for now, and after completion, you can observe an entry has been added to the registrations as well as properties (in green) and settings (in orange) for that new application:

![image](https://user-images.githubusercontent.com/18376283/215505786-ebd9e43b-b0c8-40fb-9768-5c137dff8f77.png)

The properties are the first part of artifacts I want to describe, as they are important from an analyst perspective and will reflect in the available logs:

| Property | Description |
| ----------- | ----------- |
| Display Name | This is the name of your application, nothing fancy as such |
| Application (client) ID | This is a generated ID for your application ands uniquely identifies this application |
| Object ID | This is also a generated ID for your application, but it represents an ID for this object (the application) in the current tenant |
| Directory (tenant) ID | This is your tenant ID |
| Supported account types | This tells if this application supports one or multiple tenants |

**Understanding Application ID and Object ID**

As described here above, the application ID is a unique identifier for your application, cross-tenant(s). It uniquely represents your application. 
The object ID is a unique identifier for the object, which is your application in this case, in the Azure AD tenant. See the object ID as the identifier from an Azure AD objects standpoint amongst other Azure AD objects like Users, Groups...Example for a user:

![image](https://user-images.githubusercontent.com/18376283/215521541-1e22a8f3-9265-4075-83da-6c4dc8a8dbad.png)

A few settings are also interesting to look at (all settings would be, but we need to limit the length of this article, right?):

| Property | Description |
| ----------- | ----------- |
| Owners | When an application is created by an administrative role (Global Administrator, Application Administrator etc.), it has no owner by default, this means an administrator needs to manage the application configuration. An administrator can however delegate this to someone, by assigning an owner to the application. If a non-administrator user creates an application (if such things are allowed in your tenant), he will automatically be assigned as an owner. |
| Certificate and Secrets | This is where application owners would defined secrets or certificates the application can use to authenticate to remote services: Azure AD API, Microsoft 365 or other business-related services. |
| API permissions | Applications are authorized to call APIs when they are granted permissions by users/admins as part of the consent process. |
| App Roles | App roles are custom roles to assign permissions to users or apps. The application defines and publishes the app roles and interprets them as permissions during authorization |
| Roles and administrators | Administrative roles are used for granting access for privileged actions in Azure AD. We recommend using these built-in roles for delegating access to manage broad application configuration permissions without granting access to manage other parts of Azure AD not related to application configuration. Assignable roles are roles that can be assigned here to allow managing this resourc |

Our application now has properties and settings in the target tenant. 
Any object in an Azure AD tenant however requires permissions to perform actions, how does this work with an application? 
This is where the identity part kicks in, with the infamous service principals.


### The application identity, as known as: the service principal

You have an application object and settings, but how does that application actually authenticates itself to your tenant APIs or services it needs to use? How is it able to impersonate your users? How does it differ from another application? and finally, how can this application work in another tenant?

In Azure AD, a security principal defines the access policy and permissions for a user/application in a Azure AD tenant. 
Users are represented by user principals and applications are represented by service principals.
A service principal therefore represents the **identity** of your application **inside a specific tenant**. If your application is multi-tenant (hence, allowing users from multiple tenants to use that application), it will have one service principal ID per tenant. 

The applications identities, hence service principals, are in another blade of Azure AD: Enterprise Applications (well, next to Users section of course, as service  principals are users). Albeit confusing, this is the list of service principals in your tenant and their references to the applicatiob objects they represent:

![image](https://user-images.githubusercontent.com/18376283/215535303-e1027372-eb6f-4b1a-9388-667e439b1658.png)






## Detecting Azure AD Application threats using Azure AD logs
