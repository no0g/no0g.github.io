---
layout: post
title: "FYP WAS DEPRESSING but . . ."
comments: true
description: "A summary of my Final Year Project (APU H4CKVERSITY)"
tags: "FYP CTF Pentesting"
---

## Intro . . .

To finish my BSc (Hons) Cyber Security degree. APU requires me to submit a final year project (FYP) and this post will be a summary of my FYP!   
   
The full title of my FYP is "Penetration Testing Simulation Platform for IT Security Students". It is pretty self-explanatory so, without no further ado, here it is my final year project:

## What is it?
Let me start by describing the platform i built. The platform, later will be called APU H4CKVERSITY, is a penetration testing learning platform which adopts the element of CTF that can commonly be found in a modern virtual hacking lab such as TryHackMe and HackTheBox. The main differece between APU H4CKVERSITY and other platforms is that it was built to satisfy the needs of both students and lecturers.   
   
So, not only student can learn penetration testing, but lecturers on the other hand can monitor, control and manage students inside classes in which a vulnerable machine will be deployed and attacked.

To put it simply:

> It is a gamified learning platform in which students and lecturers can deploy vulnerable machine to simulate penetration testing activity 


## Why was it built?
The idea came from my personal experience. I remembered clearly every time we had a hands-on ethical hacking class, several students and even the lecturer were facing some issues due to the heavy-load of running multiple Virtual Machines (VMs) simultaneously.
   
With the lecturer having to run multiple VMs while sharing his/her screens, the device that was used could not really handle the load. This slows down the machine and slows down the entire class learning process.

## How was it developed?
It was developed using NodeJS utilizing Express.js as web development framework. To control Virtual Machine, i used [node-virtualbox](https://www.npmjs.com/package/virtualbox) API available from NPM. For the VPN, i used OpenVPN to enable tunneled connection. Lastly, to generate keys for OpenVPN Configuration, i used Easy-RSA which is very easy to use.

## Result
The app passed all the test i defined myself (lol) and proudly, i'd say that my lecturers are excited about the platform :)

## Published Article
![Published Screenshot](https://user-images.githubusercontent.com/56182238/230714426-aa14aa07-a91a-413c-ac99-357484ae49fc.png)

[VIC21 Innovation Insights, 65-70](https://vic22.my/wp-content/uploads/2021/11/INNOVATION-INSIGHTS-SERIES-2-2021.pdf#page=77) 

[Github Repo](https://github.com/no0g/h4ckversity)

## Some Screenshots
- Landing Page
![landing](/assets/images/landing.png)
- Classes
![landing](/assets/images/class.png)
- Access Page
![landing](/assets/images/vpn.png)
## Poster
![landing](/assets/images/poster.png)

**Poster Design by: Salna Efendi**   
Check her out on:
- [Behance](http://behance.net/salna)   
- [Carrd](http://sfe.carrd.co/)
