---
layout: posts
Excerpt:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
title:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

**Phishing attacks are still a major attack vector targeting companies and individuals worldwide, and while being a constant mean to achieve initial access, techniques have always evolved over time to lure users and bypass enterprise detections. 
A recent trend in phishing campaigns is to leverage QR codes to trick users into scanning them with their mobile phones. The embedded link resolved by the QR code application will then takes the user to a credentials harvesting website for instance, or yet trigger the download of malicious software. 
There are many defenses against malicious URLs or malicious attachements in enterprise email world today, but very few solutions (as of day of writing) have the ability to analyze the content of an image or PDF file to detect usage of QRCode, resolve the URL and classify the mail accordingly.
While companies are working on resolving this shortcoming, and probably provide strong detection signatures thanks to the rise of ML models, attackers benefit of a step forward by leveraging this technique and apparent legitimate authenticaiton emails. 
In this article, I am exploring how Computer Vision (here, from Azure AI service) can help to label email attachments in an Exchange mailbox, and take action based on the results. 
The pproposed solution is exploring two scenarios: 1. Using multi-label prediction to give return the probability of an email attachment being a QR code and 2. Detecting coordinates of a QRCode inside a successfuly classified image to actually resolve the URL behind it and assess its maliciousness.
**
