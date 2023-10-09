---
layout: posts
Excerpt:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
title:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

> Phishing attacks are a prevalent threat to the security of both organizations and individuals worldwide.
> These attacks often rely on deceiving users into clicking on malicious links or opening malicious attachments that can compromise their credentials or systems.
> However, as email security solutions have become more sophisticated in detecting and blocking such attacks, phishing techniques have also evolved to evade them.
> One of the emerging trends in phishing campaigns is the use of QR codes embedded in images or PDF files to direct users to malicious websites.
> Unlike plain text URLs or attachments, QR codes are not easily analyzed by most email security solutions (as of the date of writing), and require the use of machine learning models to extract and classify their content.
> This gives the attackers an advantage in bypassing the existing defenses and gaining initial access to the target network.
> In this article, I will explore, for purely educational purposes, how computer vision can help to detect QR codes in email attachments using image classification or object detection, and how to take appropriate actions based on the results.

<div style="text-align:center">
<img src="https://i.pinimg.com/736x/2c/ef/ed/2cefed8eff6c9389d9322c1e1d6ebebc--marvel.jpg" width="400px" />
</div>

## A recap on Quishing or QRCode Phishing 

QR Code phishing, or _quishing_, is...

### References
### Mitigations

## A brief on this (educational) solution 

## Forewords

- This solution is not meant to be production-grade or used as such within an enterprise environment. It is just food-for-thought
- The solution leverages Azure Computer Vision as a ML engine due to its ease of use, but any ML solution could do the trick
- Machine Learning is not a requirement for QRCode detection and/or decoding, libraries exist to detect and decode QRCodes in images based on QRCode specification (). ML was used in a learning perspective, and also because attackers could deviate from official specifications, while having still a readable QRCode. Training a model can thus be an interesting alternative
- The code is intentionnally written in C# for self-learning purposes, but I am not a professional C# developer (neither a developer anymore): the code is definitely not scalable neither security-proof. A simple REST client has been preferred over Azure SDK for Graph API but both could be used interchangeably.  
- The model used for labeling and detecting QR Codes was trained on a minimalistic dataset, and a professional model should have thousands of training data and several iterations to optimise model perfornances
- The solution is a standalone C# program acting on a specific mailbox, a few ideas to leverage it (for learning/testing purposes) are: (1) Monitor a quarantine mailbox (example taken in this article), (2) Add-on to Exchange anti-spam policies or Office 365 Safe Links, (3) A logic app which would act based on the results, or create an incident in a SIEM, or yet (4) a simple scheduled task on a server
- The model can be bypassed easily, for instance using PDF files amongst many other things, the purpose is educationnal


## Implementation Details

### The ML model 

### The code 

## How to integrate?

## Features to explore 
