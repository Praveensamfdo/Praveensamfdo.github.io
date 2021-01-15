---
layout: post
title: "RSA simply explained"
date: 2018-08-03 02:40:00 +0000
tags: security encryption
description: "A brief explanation of RSA algorithm"
---
RSA is an asymmetric key cryptosystem, where a public key and a private key is involved.

RSA is basically used to achieve two tasks.
1. To achieve public-key encryption
1. To support digital signature

Let's discuss RSA based on the following scenario for the sake of clarity.

![](/assets/post_images/rsa_overall.png)

## 1. Public-key encryption explained

Public-key encryption is used to encrypt the sender’s message.
Let’s assume that Will wants to send a message to Smith. Then the encryption operation can be described as follows.

1. Smith(receiver) sends his public key to Will.
1. Will(sender) uses that key to encrypt the message.
1. Encrypted message is sent to Smith.
1. Smith uses his private key to decrypt the message.

![](/assets/post_images/rsa_1.png)

Hence it can be clearly seen that, possessing of Smith’s public key can enable any sender to send an encrypted message to Smith.

![](/assets/post_images/rsa_2.png)

This is the public-key encryption in a nutshell.
One can identify some major vulnerabilities when analyzing the above scenario.

1. Authentication problem – Receiver can’t guarantee that the message he received is via the claimed sender.
1. Integrity problem – An attacker can tamper an encrypted message, as there is no way for receiver to check the integrity of the incoming message.

To address the above two issues, there is a concept called **Digital Signature**.

## 2. Digital Signature Explained

Let’s assume that Will wants to send a message to Smith. Then Digital Signature is achieved as follows.


1. Will(sender) sends his public key to Smith.
1. Will generates a digest by hashing his message using a hashing algorithm (Ex: MD5, SHA-256).
1. Will uses his private key to encrypt the digest and generates the Digital Signature.
1. Both the original message and the Digital Signature is sent to the receiver.
1. Upon arrival, Smith tries to decrypt the digital signature using the claimed sender’s public key.
	-  If he succeeds to encrypt then he can be sure that the message has been originated from the claimed sender.
	-  If not, the message has been originated from some other source.
1. Smith hashes the received message and compares it with the received digest decrypted from step 05.
	- If two digests match, then the received message has not been tampered during transmission.
	- If they don’t match, then the message has been tampered.

![](/assets/post_images/rsa_2.png)

The user authentication and message integrity are achieved via step 05 and 06 respectively.


