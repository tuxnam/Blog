---
layout: posts
Excerpt:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
title:  "Quish Me If You Can: Detect QR Code phishing emails in Exchange using computer vision"
last_modified_at: 20@#-01-30T13:59:57-04:00
---

*Phishing attacks are still a prevalent threat to the security of both organizations and individuals worldwide. These attacks often rely on deceiving users into clicking on malicious links or opening malicious attachments to harvest credentials or authentication tokens. However, email security solutions have become more sophisticated in detecting and blocking such attacks. Moreover, Microsoft’s decision to block Office macros by default has made it harder for attackers to execute malicious code. Therefore, phishing techniques have also evolved to evade them.

One of the emerging trends in phishing campaigns is the use of QR codes embedded in images or PDF files to direct users to malicious websites. Unlike plain text URLs or attachments, QR codes are not easily filtered out by most email security solutions. This problematics was the occasion for me to experiment with Machine Learning as this is an interesting use case for the recent democratization of ML models to detect, label or describe image content. There are many ways to leverage machine learning models, but they can be complex. Starting with a concrete modern security issue can hopefully help alleviate some of the complexity and give initial thoughts for further exploration.

In this article, I will explore, for purely educational purposes, how computer vision can help to detect QR codes in email attachments using image classification or object detection. These are two common computer vision tasks that can be performed by various machine learning models. The post is accompanied with a proof-of-concept written in C# which can be adapted to your own (experimentation) needs.*

<div style="text-align:center">
  <img src="https://github.com/tuxnam/Blog/assets/18376283/7e5d2374-e6bf-48f7-bde9-f404fddfbddd" width="400px" />
</div>

## A recap on Quishing or QRCode Phishing 

Here is my rewritten paragraph for you:

Quishing, or QR Code phishing, is a type of phishing attack that became more popular since mid-2023. It uses a QR Code in an email to trick the victim into scanning it with their mobile device. The email pretends to be from a trusted company and tries to persuade the victim to act quickly or emotionally (for example, by offering a reward, asking for a password reset, mentioning a police statement, etc.).
When the victim scans the QR Code, they are taken (often through several redirections to avoid detection, using open redirects on trusted domains) to a malicious website that steals their credentials or other personal data. 
The attacker can then use this data to compromise the victim’s account or device.

This article does not aim to provide a detailed analysis of the technique, the countermeasures, or the observed campaigns, but rather to reflect on opportunities offered by ML for modern phishing attacks. 
You can find some useful references on the attack, its detection, and its prevention below. However, remember that Quishing is just one form of phishing and user education is still the most important factor. 
Attackers will always find new ways to bypass detection, whether it is with QR Codes today or something else tomorrow. You can also check <a href="https://www.bing.com/ck/a?!&&p=2d1250c2f19ff83cJmltdHM9MTY5NzI0MTYwMCZpZ3VpZD0wNTk2MjcxMi1kNmYwLTZhMGYtMzUyYi0zNDhjZDdiODZiZjQmaW5zaWQ9NTIxMw&ptn=3&hsh=3&fclid=05962712-d6f0-6a0f-352b-348cd7b86bf4&psq=phishing+mitigations+microsoft&u=a1aHR0cHM6Ly93d3cubWljcm9zb2Z0LmNvbS9lbi11cy9zZWN1cml0eS9ibG9nLzIwMjAvMDgvMjUvZGV0ZWN0LW1pdGlnYXRlLXBoaXNoaW5nLXJpc2tzLW1pY3Jvc29mdC1zZWN1cml0eS8&ntb=1">this link</a> for some general phishing prevention tips. 

Finally, note that this article focuses on Quishing but there are other risks associated with scanning unknown QR Codes (such as using a vulnerable QR Code app), which you can learn more about by researching QR Code fuzzing.

Note: another aspect that I think deserves more attention is device management. It is crucial to ensure that mobile devices are managed by the company (if they contain work-related data), and block access to known malicious URLs as well as detect suspicious authentication attempts from unmanaged or unfamiliar devices (which could indicate a successful phishing attack).

### References on Quishing

There are numerous references on the subject, here are some of them:

Any.run - New phishing tactics - https://any.run/cybersecurity-blog/new-phishing-tactics/
Perception-point - QR Code Red: Quishing Attacks and How to Prevent Them - https://perception-point.io/blog/qr-code-red-quishing-attacks-and-how-to-prevent-them/
Malware Bytes - Targeted phishing campaigns - https://www.malwarebytes.com/blog/news/2023/08/qr-codes-deployed-in-targeted-phishing-campaigns
Bleeping Computer - https://www.bleepingcomputer.com/news/security/phishing-attacks-use-qr-codes-to-steal-banking-credentials/
Identify Quishing Emails using Sentinel - https://rodtrent.substack.com/p/microsoft-sentinel-soc-101-how-to-b94

## Why Machine Learning?

There are multiple existing ways to attempt to detect phishing emails which could apply to quishing, a non-exhaustive (and complementary) list:
- DMARC, DKIM, SPF checks
- Keywords in the content or the header of the email (password, MFA, urgent, payment...) 
- Threat intelligence IOCs (IP addresses, domain names, behavior, subject...)
- Recently registered domains
- Malicious or suspicious attachments or links
- Heuristics and models based on multiple patterns
- Attachments analysis
- URLs scanning
- ...

Most modern email security solutions generally offer a detonation capability for emails, in addition to any of the above detections, which allows them to analyze emails in a sandbox for further investigation. Detonation can trigger attachments, follow URL redirections, examine email content and compare multiple signatures that are usually hosted, maintained and updated by the security provider. However, QR Codes are simple images that do not pose any obvious threat (from a signature perspective) and that most security solutions, at the time of writing, are not able to detect efficiently. <br />

The latter is not entierly correct, a few existing vendors will detect Quishing based on the above bullets (mainly IOCs), some of them are also trying to detect and read QR Codes already in attachments, but this has some limitations. 
One of them is that QRCode detection generally works by decoding an image based on QRCode ISO standard: <a href='https://www.iso.org/standard/62021.html'>ISO 18004:2015</a> The standard defines the requirements for the symbology known as QR Code. It specifies the QR Code characteristics, data character encoding methods, symbol formats, dimensional characteristics, error correction rules, reference decoding algorithm, production quality requirements, and user-selectable application parameters. It is however possible to deviate slighly from these specifications and still having a QRCode which most QRCode application readers will be able to decode, or find ways to be undetected by most QRCode reading libraries but still being able to be read by a mobile device camera. 

For instance, take a random QRCode analyzer online, it will be able to decode most of the QRCodes you would challenge him too and return the encoded string, but what about this one:

<p>
<img src="https://github.com/tuxnam/Blog/assets/18376283/d3644f52-e12e-4e2e-b69c-9c0806ade701" width="198px" />&nbsp;&nbsp;<img src="https://github.com/tuxnam/Blog/assets/18376283/53beda1c-0964-4f96-85ec-6657f42c0de1" width="650px" />
</p>

And, yet, scanning the same image with your mobile device result in the QRCode being decoded:

<img src="https://github.com/tuxnam/Blog/assets/18376283/1019d33e-9169-4286-ab07-f6b1e0501756" width="225px" />
<br />

This is just a simple, random example I found in a few minutes of testing, so imagine what clever phishers could come up with? 

**This is where Machine Learning can come handy!**

Machine learning is not a magic, bullet-proof solution either, but it offers an interesting approach based on probabilistic labeling (is that even a thing? Data scientists, if your lost your way, be gentle here), rather than formal specifications (ISO), and the more the model is trained (based om regular QR codes, QR codes from known attack campaigns), the more efficient it will be at validating the presence of a QRCode in a picture, in a complementary way and next to the above existing phishing detection patterns. 

### What kin of Machine Learning?

I am not an ML/AI expert and the idea was also to learn and leverage available tools for the domain I know, security. 
Nowadays, with the rise of ML tools, it is easy to have access to powerful models which have been "tuned" for purpose (i.e.: algorithms chosen wisely and parametrized), you just basically have to know which type to use, create and train a model. 
The models used behind computer vision are based on deep learning, and more specifically techniques such as transfer learning, mmodel composition, object detection, or yet, few-shot learning (behind others). 

While building such a model from scratch is complex, the availability of ML products today makes it easier to do simple tasks, such as the one we are interested in: detecting the presence of QRCode inside an email. 

## The approach 

The idea is to be able to 'scan' one or multiple Exchange mailboxes (through an Exchange search filter, or by monitoring a quarantine shared inbox or yet VIP users accounts mailboxes for instance) to find potential Quishing targeted emails. 

What I tested in this case is using <a href="https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fexchange%2Fsecurity-and-compliance%2Fmail-flow-rules%2Finspect-message-attachments&data=05%7C01%7Cgbenats%40microsoft.com%7C9d12e40f00e5422d5eb908dbc7138454%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C638322659099496432%7CUnknown%7CTWFpbGZsb3d8eyJWIjoiMC4wLjAwMDAiLCJQIjoiV2luMzIiLCJBTiI6Ik1haWwiLCJXVCI6Mn0%3D%7C3000%7C%7C%7C&sdata=0ijKV7Es8xq7QRgHGOheFYJL99byLhl5ukBQlCeBxTU%3D&reserved=0">email flow rules to inspect message attachments in Exchange Online</a>, identify attachments which could be QR codes and *submit them to a computer vision model*.  
The proof-of-concept includes actually decoding the QRCode which allows to scan the decoded URLs through usual suspects such as VirusTotal, MDO Safe Links or Hybrid Analysis for instance. 
The decoding steps is quite basic and limited to the drawback explained above (ISO deviation), it could fail to decode the QR Code. 
We can imagine also leveraging ML models to make a 'clean' QRCode out of a 'non-compliant' one, but that would be another research on its own, and is way outside of my current ML skills.

In this proof-of-concept, I decided to use **Azure Computer Vision service**, but the same approach would work with Google Vision AI, AWS Rekognition or any other service of your choosing. You could even build your own service using TensorFLow and the like. 

The proof-of-concept is there solely for learning purpose and could be extended in many ways:
- Automated actions on the identified suspicious emails
- Integration in a serverless context to be leveraged for instance with Azure Logic Apps and a SIEM, as a SOAR playbook to analyze a specific email entity (think of users reported emails for instance)
- ...

**Notes**
- The PoC was written in C# (just because I wanted to "learn" C# and was a bit bored of Python) but it can easily be done in any other language (this leverages REST APIs mostly) and for instance use in a Jupyter Notebook for threat hunting purposes.
- The code is not production-ready, 'clean' or bullet-proof - it exists for the sole purpose of learning.
- A deep-learning model is as efficient as the number of labeled objects you used to train it. I did train the model only with a few dozens images.
- You could imagine re-using the results of each prediction to feed the model in iterations.
- The most recent Computer Vision services now combine Vision AI and Large Language Models (LLMs), which will descirbe an image without having to pre-train a model yourself: open-world classification (being trained based on biilions of images) vs closed world training methods (trained by your own input)
 
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

With that, I had my prediction API and the corresponding trained model, I could start building a program to target Exchange emails and see the results. 

### The proof-of-concept (C#)

The program's goal is to connect to Exchange through <a href="https://developer.microsoft.com/en-us/graph">Graph API</a>, and analyze emails in a mailbox, a list of mailboxes or resulting from a <a href="https://learn.microsoft.com/en-us/graph/filter-query-parameter?tabs=http">filter query</a>. 
All emails with attachments are filtered out to keep only images content-types. 
Resulting attachments are downloaded, send to the Computer Vision prediction model built above, and based on the results (probably threshold defined in a configuration file), defined as suspicious or not. 
Suspicious attachments are further decoded by a QRCode library and the resulting URL is added to the resulting analysis. 
The program outputs a _.csv_ file with all the emails analyzed which contains a potentially malicious QR Code. 

The permissions model to Graph API and Exchange works by using a Service Principal (Azure AD application) having permissions to read emails, and the client credentials grant type (https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow).
As having a read capability to all mailboxes of an organization within a single application could be dangerous, it is advised to add an <a hraf="https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access">Exchange policy</a> limiting the permissions of the application. In my case, the application can only read from _quarantine@guillaumeben.xyz_. 

Here is a sample test results:

#### 1. Send malicious and non-malicious emails to a test mailbox

These emails have either QRCodes as attachments, "safe"attachments, no attachments or embedded QRCodes. 

![image](https://github.com/tuxnam/Blog/assets/18376283/24bfd656-7f86-48e1-aad0-094b89e8291e)

#### 2. Configure the program 

The PoC uses a simple configuration file for all things required which is described in the GitHub repositor's README.md file.
  
![image](https://github.com/tuxnam/Blog/assets/18376283/00012e8f-bd49-4564-8290-31037e8a5bfa)

#### 3. Run

![image](https://github.com/tuxnam/Blog/assets/18376283/15bb6ba1-fa2f-40a8-8fda-abb42f7f5341)

![image](https://github.com/tuxnam/Blog/assets/18376283/f9218a37-1fa0-45e7-9399-99c62b290e87)


#### 4. Analyze Output

The program outputs the downloaded attachment and a corresponding csv file with the results. You notice both inline attachments and enclosed attachments are analyzed. Most Quishing emails seen to date are using inline attachments. 

![image](https://github.com/tuxnam/Blog/assets/18376283/6f90a396-57dc-467e-b154-c02130f68216)

![image](https://github.com/tuxnam/Blog/assets/18376283/53df5dc1-06e6-4895-b6d5-e69895e766d8)


## Conclusion 

Exploring the opportunities offered Machine Learning in concreate security threats can help alleviate some of the fuzz around these techniaues.
We explored how a "modern" phishing technique that uses QRCodes to lure victims to malicious websites can be detected using simple computer vision models. 
Quishing is a serious threat that orgnanizations have to face with today and ML cam help alleviate the inherent difficulty or products limitations in detecting such threats.
However, there are limitations and challenges to the approach, such as the usual suspects (false positives, false negatives) and how well the model is trained. 
Readers should also keep in mind that Quishing is just one of the many forms of phishing, attackers will keep coming with new and clever techniques over time. Despite a need to have strong detections and mitigations in place, Users vigileance will always be the first (or last) line of defense. 
