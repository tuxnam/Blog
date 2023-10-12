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

QR Codes are simple images without any apparent threat (from a signature perspective) and for which most security solutions, at time of writing, are not able to detect efficiently. <br />

**Is this really the case? **<br />
A lot of existing vendors will detect Quishing based on the above bullets (mainly IOCs), some of them are also trying to read QR Codes already in attachments, but this can have some limitations. 
One of these limitations is that QRCode detection generally works by decoding the image and match it to the QRCode ISO standard: The ISO/IEC 18004:2015 standard defines the requirements for the symbology known as QR Code. It specifies the QR Code symbology characteristics, data character encoding methods, symbol formats, dimensional characteristics, error correction rules, reference decoding algorithm, production quality requirements, and user-selectable application parameters. It is however possible to deviate slighly from these specifications and still having a mobile-readable QRCode, or find ways to be undetected by most QRCode reading libraries but still being able to be read by a mobile device camera. 

For instance, take a random QRCode analyzer online, it will be able to decode most of the QRCodes you would challenge him too and return the encoded string, but what about this one:

<img src="https://github.com/tuxnam/Blog/assets/18376283/7d5ae045-6ae0-45a2-8b89-886e3637c1c1" width="150px" />

<img src="https://github.com/tuxnam/Blog/assets/18376283/53beda1c-0964-4f96-85ec-6657f42c0de1" width="600px" />

And, yet, with the mobile device:

<img src="https://github.com/tuxnam/Blog/assets/18376283/1019d33e-9169-4286-ab07-f6b1e0501756" width="300px" />

This is just a simple, random example I tested in 2 minutes, so imagine what clever phishers could come up with? 

This is where Machine Learning can come handy. 
Machine learning is not a magic, bullet-proof solution either, but it offers a novel approach based on probabilistic determination (is that even a thing? Mathematicians, be gentle here), rather than specifications, and the more the model is trained (based om regular QR codes, QR codes from known attack campaigns), the more efficient it will be at validating the presence of a QR Code in a picture, in a complementary way and next to the above existing phishing detection patterns. 

### What do I mean by Machine Learning?

I am not an ML/AI expert and the idea was also to learn and leverage these tools for the domain I know, security. Nowadays, with the avenant of ML tools, it is easy to have access to powerful models which have already been "tweaked" (i.e.: parametrized) to fit the most common use cases, you just basically have to train them and use them. 
The models used behind computer vision are based on deep learning, and more specifically techniques such as transfer learning, mmodel composition, object detection, or yet, few-shot learning behind others. 
While building such a model from scratch is complex and requires very specific skills, the availability of tools today makes it easier to do simple tasks, such as the one we are interested in: detecting the presence of QR Code inside a picture file. 

## The approach 

The idea is to be able to 'scan' one or multiple Exchange mailboxes (through an Exhcange search filter, or by monitoring a quarantine shared inbox or yet VIP users accounts for instance), identify attachments which could be QR codes, submit them to a computer vision model and export the results for further analysis. The proof-of-concept includes actually decoding the QRCode which allows to scan the decoded URLs through usual suspects such as VirusTotal, MDO Safe Links or Hybrid Analysis for instance. The decoding steps is quite basic and limited to the drawback explained above, it could fail to decode the QR Code. Here, however, we could imagine also leveraging ML models to make a 'clean' QRCode out of a 'non-compliant' one, but that would be another research on its own, and way outside of my non-existent ML skills.
In this proof-of-concept, I decided to use Azure Computer Vision service, but the same approach would work with Google Vision AI, AWS Rekognition or any other service of your choosing. You could even build your own service using TensorFLow and the like. 

The proof-of-concept is there solely for learning purpose and could be extended in many ways:
- Automated actions on the identified suspicious emails
- Integration in a serverless context to be leveraged for instance with Azure Logic Apps and a SIEM, as a SOAR playbook to analyze a specific email entity (think of users reported emails for instance)
- ...

**Notes**
- The PoC was written in C# just because I wanted to learn C# but it can easily be done in Python and for instance use in a Jupyter Notebook for threat hunting purposes
- The code is not production-ready, 'clean' or bullet-proof - it exists for the sole purpose of learning
- A deep-learning model is as efficient as the number of resources you used to train it. I did train the model only with a few dozens of labelled images.
- You could imagine re-using the results of the PoC to feed the model in a new learning iteration
- The most recent Computer Vision services now combine Vision AI and LLMs, which will descirbe an image without having to pre-train a model yourself: open-world classification (being trained based on biilions of images) vs closed world training methods (trained by your own input)
 
### The model

Building the model is easy, I just followed Azure Computer Vision documentation: https://learn.microsoft.com/en-us/azure/ai-services/custom-vision-service/.
The Computer Vision project built therefore contains two parts: a classification model and a prediction model. You can leverage either the GUI (Azure Vision Studio) or the SDK to consume both models. 
The first step was to train a create a project, train the model with labeled data (QR Codes in our case) and train it. This is using multi-labeling classification between two labels: QRCODE or NOCODE.
Once the model is trained, it can be used for predictions (hear, a propbability) of an image being labeled as a QRCODE or not. 

![image](https://github.com/tuxnam/Blog/assets/18376283/6c335d87-3ad3-41e9-9b63-247b2412bd28)

When you train a model, you are supposed to do several iterations, assess your model performances, review parameters or configuration and keep tuning it until you obtain satisfying metrics: precision and recall level mostly (similar to detection engineering in security if you think about it!) or yet Average Precision (AP) or Mean Average Precision (MAP). 
In this case, I kept it simple with a single iteration and only a few images, just for the sake of proof-of-concept:

![image](https://github.com/tuxnam/Blog/assets/18376283/292ce036-2a65-46d4-bda1-c9f1aa1224e7)

Once the model was trained and I was satisfied (ok, I was quickly satisfied in this case...) of the performances, I simply could leverage the Prediction API to identify the label of a new image (you can guess I debugged my code over and over with the same sample picture).

![image](https://github.com/tuxnam/Blog/assets/18376283/42658bbc-e7b4-42eb-86a1-c411a19f10b3)

Ok, with that, I had my prediction API and our model, I could start building a program to target Exchange emails and see the results. 

### The proof-of-concept (C#)



## Going further



## Forewords

- This solution is not meant to be production-grade or used as such within an enterprise environment. It is just food-for-thought
- The solution leverages Azure Computer Vision as a ML engine due to its ease of use, but any ML solution could do the trick
- Machine Learning is not a requirement for QRCode detection and/or decoding, libraries exist to detect and decode QRCodes in images based on QRCode specification (). ML was used in a learning perspective, and also because attackers could deviate from official specifications, while having still a readable QRCode. Training a model can thus be an interesting alternative
- The code is intentionnally written in C# for self-learning purposes, but I am not a professional C# developer (neither a developer anymore): the code is definitely not scalable neither security-proof. A simple REST client has been preferred over Azure SDK for Graph API but both could be used interchangeably.  
- The model used for labeling and detecting QR Codes was trained on a minimalistic dataset, and a professional model should have thousands of training data and several iterations to optimise model perfornances
- The solution is a standalone C# program acting on a specific mailbox, a few ideas to leverage it (for learning/testing purposes) are: (1) Monitor a quarantine mailbox (example taken in this article), (2) Add-on to Exchange anti-spam policies or Office 365 Safe Links, (3) A logic app which would act based on the results, or create an incident in a SIEM, or yet (4) a simple scheduled task on a server
- The model can be bypassed easily, for instance using PDF files amongst many other things, the purpose is educationnal

