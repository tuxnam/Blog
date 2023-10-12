---
layout: posts
Excerpt:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
title:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

> Phishing attacks are still a prevalent threat to the security of both organizations and individuals worldwide.
> These attacks often rely on deceiving users into clicking on malicious links or opening malicious attachments to harvest credentials or <a href="https://www.microsoft.com/en-us/security/blog/2022/07/12/from-cookie-theft-to-bec-attackers-use-aitm-phishing-sites-as-entry-point-to-further-financial-fraud/">authentication tokens</a>.
> However, as email security solutions have become more sophisticated in detecting and blocking such attacks or following Microosft's decision to <a href="https://learn.microsoft.com/en-us/deployoffice/security/internet-macros-blocked">block Office macros by default</a>, phishing techniques have also evolved to evade them.
> One of the emerging trends in phishing campaigns is the use of QR codes embedded in images or PDF files to direct users to malicious websites.
> Unlike plain text URLs or attachments, QR codes are not easily analyzed by most email security solutions, and are an interesting learning use case for the recent democratization of machine learning models to extract and classify inage content.
> Ways to leverage machine learning models are numerous but sometimes complex. Starting with a concrete modern security issue can hopefully help alleviate some of the complexity and give initial thoughts for further exploration.
> In this article, I will explore, for purely educational purposes, how computer vision can help to detect QR codes in email attachments using image classification or object detection. I have written a proof-of-concept in C# which can be adapted to your own (experimentation) needs.

<div style="text-align:center">
<img src="https://i.pinimg.com/736x/2c/ef/ed/2cefed8eff6c9389d9322c1e1d6ebebc--marvel.jpg" width="400px" />
</div>

## A recap on Quishing or QRCode Phishing 

QR Code phishing, or _quishing_, is a recent (campaigns have been seen rising around mid-2023 and keep going) phishing attack technique whereby an attacker sends an email to its victim and try to lure them into scanning an embdeed QR Code using their mobile device.
Like for regular targeted phishing, the email template is generally trying to spoof a "legitimate" company and indicates a sense of urgency or a way to get to the reader's feelings (password reset, financial gain, police statement, secret document...).
By scanning the QR Code, users will be redirected (more often than not, using multiple redirection layers to bypass endpoint or mobile protections) to a fraudulent website to harvest their credentials or any sensitive information which could be useful to the attacker for initial compromise. 
The goal of this article is not to ellaborate on the technique, the mitigations or the observed campaigns but a small recap is always welcome. I added here below some interesting references to the attack, analysis and existing mitigations.

### References
### Mitigations

## Why Machine Learning?

There are multiple existing (complementary) ways to attempt to detect Quishing emails, a non-exhaustive list:
-
-
-

However, there are limitations in the above methods which are a good use case for machine learning:

-
-

## The approach 

## The proof-of-concept (C#)

## Going further



## Forewords

- This solution is not meant to be production-grade or used as such within an enterprise environment. It is just food-for-thought
- The solution leverages Azure Computer Vision as a ML engine due to its ease of use, but any ML solution could do the trick
- Machine Learning is not a requirement for QRCode detection and/or decoding, libraries exist to detect and decode QRCodes in images based on QRCode specification (). ML was used in a learning perspective, and also because attackers could deviate from official specifications, while having still a readable QRCode. Training a model can thus be an interesting alternative
- The code is intentionnally written in C# for self-learning purposes, but I am not a professional C# developer (neither a developer anymore): the code is definitely not scalable neither security-proof. A simple REST client has been preferred over Azure SDK for Graph API but both could be used interchangeably.  
- The model used for labeling and detecting QR Codes was trained on a minimalistic dataset, and a professional model should have thousands of training data and several iterations to optimise model perfornances
- The solution is a standalone C# program acting on a specific mailbox, a few ideas to leverage it (for learning/testing purposes) are: (1) Monitor a quarantine mailbox (example taken in this article), (2) Add-on to Exchange anti-spam policies or Office 365 Safe Links, (3) A logic app which would act based on the results, or create an incident in a SIEM, or yet (4) a simple scheduled task on a server
- The model can be bypassed easily, for instance using PDF files amongst many other things, the purpose is educationnal

