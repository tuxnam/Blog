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

<div style="text-align:center">
<img src="https://user-images.githubusercontent.com/18376283/224830681-ff326f42-5966-4c50-a353-0f5b55df62da.png" width="400px" />
</div>

## Forewords

This solution is not meant to be production-grade or used as such within an enterprise environment. The idea was to explore customer-ready computer vision solutions available to users today, and leverage it in a concrete example which could give food for thoughts on people actually trying to fight that new phishing trend. 
A small disclaimer is therefore necessary:
- The code is intentionnally using C# for self-learning purposes, but I am not a professional C# developer. The code indeed lacks a lot of error and exceptions handling, is not scalable and the overall approach lacks some strong security mechanismns around usage of APIs.
- The model used for labeling and detecting QR Codes was trained on a minimalistic dataset, and a professional model should of course have thousands of training data and evaluate model perfornances accordingly
- The solution is a standalone C# program acting on a specific mailbox: the idea is for instance to leverage such a solution along with Safe Link () or to monitor a quarantine mailbox before taking action on an email
- The model can easily be bypassed easily, for instance using PDF files amonst many other things, the purpose of this exercise is educationnal

## A quick recap on Quishing techniques

### What is it?

The goal of this article is not to cover the subject of Quishing, its mitigations and the recent campaigns many companies have suffered from, as there are already many article on the subject (see section References). However, it is always good to do a recap. 

### How does it work?
### A few examples
### Additional indicators 
### References

## The proposed solution

## Details of the approach 

## Repository 

## Ideas for improvment 
