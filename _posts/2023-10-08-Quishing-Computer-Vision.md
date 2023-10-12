---
layout: posts
Excerpt:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
title:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

*Phishing attacks are still a prevalent threat to the security of both organizations and individuals worldwide.
These attacks often rely on deceiving users into clicking on malicious links or opening malicious attachments to harvest credentials or <a href="https://www.microsoft.com/en-us/security/blog/2022/07/12/from-cookie-theft-to-bec-attackers-use-aitm-phishing-sites-as-entry-point-to-further-financial-fraud/">authentication tokens</a>.
However, as email security solutions have become more sophisticated in detecting and blocking such attacks or following Microosft's decision to <a href="https://learn.microsoft.com/en-us/deployoffice/security/internet-macros-blocked">block Office macros by default</a>, phishing techniques have also evolved to evade them.
One of the emerging trends in phishing campaigns is the use of QR codes embedded in images or PDF files to direct users to malicious websites.
Unlike plain text URLs or attachments, QR codes are not easily analyzed by most email security solutions, and are an interesting learning use case for the recent democratization of machine learning models to extract and classify inage content.
Ways to leverage machine learning models are numerous but sometimes complex. Starting with a concrete modern security issue can hopefully help alleviate some of the complexity and give initial thoughts for further exploration.
In this article, I will explore, for purely educational purposes, how computer vision can help to detect QR codes in email attachments using image classification or object detection. I have written a proof-of-concept in C# which can be adapted to your own (experimentation) needs.*

<div style="text-align:center">
  <img src="https://github.com/tuxnam/Blog/assets/18376283/7e5d2374-e6bf-48f7-bde9-f404fddfbddd" width="400px" />
</div>

## A recap on Quishing or QRCode Phishing 

Quishing, or QR Code phishing, is a phishing attack technique that emerged around mid-2023 and has been increasing ever since. It involves an attacker sending an email to a victim and enticing them to scan a QR Code embedded in the email using their mobile device. 
The email usually mimics a legitimate company and creates a sense of urgency or appeals to the victim’s emotions (such as password reset, financial gain, police statement, secret document, etc.). 
When the victim scans the QR Code, they are redirected (often through multiple layers of redirection to evade endpoint or mobile protections, with open redirects for instance) to a malicious website that steals their credentials or other sensitive information. 
The attacker can then leverage this information for initial compromise. <br />
The purpose of this article is not to explain the technique, the mitigations, or the observed campaigns in detail, but it is worth starting with a brief recap. 
Below are some interesting references on the attack, its detection, and its prevention.

### References

Any.run - New phishing tactics - https://any.run/cybersecurity-blog/new-phishing-tactics/
Perception-point - QR Code Red: Quishing Attacks and How to Prevent Them - https://perception-point.io/blog/qr-code-red-quishing-attacks-and-how-to-prevent-them/
Malware Bytes - Targeted phishing campaigns - https://www.malwarebytes.com/blog/news/2023/08/qr-codes-deployed-in-targeted-phishing-campaigns
Bleeping Computer - https://www.bleepingcomputer.com/news/security/phishing-attacks-use-qr-codes-to-steal-banking-credentials/


## Why Machine Learning?

There are multiple existing (complementary) ways to attempt to detect phishing emails which could apply to quishing, a non-exhaustive list:
- DMARC, DKIM, SPF checks - However a lot of targeted phishing emails will have valid signatures
- Keywords in the content or the header of the email (password, MFA, urgent, payment...) 
- Threat intelligence IOCs (IP addresses, domain names, behavior, subject...)
- Recently registered domains exclusion
- Malicious or suspicious attachments or links
- Heuristics and models based on multiple patterns

Most modern email security solutions generally offer a *detonating* capability for emails which do not match, or complementary to, any of the above pattern in a sandbox for further analysis. Dwtonation allows to trigger attachments, investigate URL redirections, analyze email content and match multiple signatures generally hosted and maintained by the security provider. <br />

QR Codes are simple images without any apparent threat (from a signature perspective) and for which most security solutions, at time of writing, are not able to detect efficiently. 
A lot of existing vendors will detect Quishing based on the above bullets (mainly IOCs), some of them are also trying to read QR Codes already in attachments, but these have some limitations. 
One of these limitations is that QRCode detection generally works by decoding the image and match it to the QRCode ISO standard: 
The ISO/IEC 18004:2015 standard defines the requirements for the symbology known as QR Code. It specifies the QR Code symbology characteristics, data character encoding methods, symbol formats, dimensional characteristics, error correction rules, reference decoding algorithm, production quality requirements, and user-selectable application parameters. It is however possible to deviate slighly from these specifications and still having a mobile-readable QRCode, or find ways to be undetected by most QRCode reading libraries but still being able to be read by a mobile device camera. This is where Machine Learning can come handy. 

Machine learning is not a magic, bullet-proof solution either, but it offers a novel approach based on probabilistic determination, rather than specifications, and the more the model is trained (based om regular QR codes, QR codes from known attack campaigns), the more efficient it will be at validating the presence of a QR Code in a picture. 

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

