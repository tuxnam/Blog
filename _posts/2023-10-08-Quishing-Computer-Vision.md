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

## Forewords

This solution is not meant to be production-grade or used as such within an enterprise environment. The idea was to explore customer-ready computer vision solutions available to users today, and leverage it in a concrete example which could give food for thoughts on people actually trying to fight that new phishing trend. 
A small disclaimer is therefore necessary:
- The code is intentionnally using C# for self-learning purposes, but I am not a professional C# developer. The code indeed lacks a lot of error and exceptions handling, is not scalable and the overall approach lacks some strong security mechanismns around usage of APIs.
- The model used for labeling and detecting QR Codes was trained on a minimalistic dataset, and a professional model should of course have thousands of training data and evaluate model perfornances accordingly
- The solution is a standalone C# program acting on a specific mailbox: the idea is for instance to leverage such a solution along with Safe Link () or to monitor a quarantine mailbox before taking action on an email
- The model can easily be bypassed easily, for instance using PDF files amonst many other things, the purpose of this exercise is educationnal

The pproposed solution is exploring two scenarios: 1. Using multi-label prediction to give return the probability of an email attachment being a QR code and 2. Detecting coordinates of a QRCode inside a successfuly classified image to actually resolve the URL behind it and assess its maliciousness.


## A quick recap on Quishing techniques

The embedded link resolved by the QR code application will then takes the user to a credentials harvesting website for instance, or yet trigger the download of malicious software. 

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
