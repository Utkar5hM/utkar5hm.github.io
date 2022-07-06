---
title: Exfiltration Techniques - Notes
date: 2022-07-01 00:00:00 +530
categories: [ Cyber Security Notes, Exfiltration]
tags: [Notes, Pentesting, Exfiltration, Red Team]     # TAG names should always be lowercase
---

A common method for exfiltrating data is to smuggle it out within a harmless protocol, usually encoded. For example, DNS is often used to (relatively) quietly exfiltrate data. HTTPS tends to be a good option as the data will outright be encrypted before egress takes place. ICMP can be used to (very slowly) get the data out of the network. DNS-over-HTTPS is superb for data exfiltration, and even email is often used.  

In a real world situation an attacker will be looking to exfiltrate data as quietly as possible as there may be an Intrusion Detection System active on the compromised network which would alert the network administrators to a breach should the data be detected. For this reason an attacker is unlikely to use protocols as simple as FTP, TFTP, SMB or HTTP; however, in an unmonitored network these are still good options for moving files around.

It's worth noting that most command and control (C2) frameworks come with options to quietly exfiltrate data. Practically speaking, this is likely how a bad actor would be exfiltrating data, so it's worth keeping up to date with the current "standards" used by the various frameworks. There are also plenty of standalone tools available to automate sending and receiving obfuscated data.

As extra reading, [PentestPartners](https://www.pentestpartners.com/) have a superb [blog post](https://www.pentestpartners.com/security-blog/data-exfiltration-techniques/) on this topic.
