---
layout: post
title: HTB Writeup Writeup!
date: '2019-11-06 15:56:37'
tags:
- hack-the-box
---

# Enumeration

Starting with Autorecon and analysing the output we find a bunch of the tests failed with _connection denied_ after initially determining that ports 22 and 80 were open. Trying to connect to port 80 in a browser fails too. After a waiting a bit the text on the website suggests some kind of DoS protection is in place, hence the tools failing.

Looking through the Nmap &nbsp;output we can see that /writeup exists on the webserver. You can also find this by looking at /robots.txt.

A quick read through provides nothing much of interest, but looking at the page source we can see a CMS is in use.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-15h21m58s001.png" class="kg-image"></figure>

Reading into this product it seems the admin URL is at /admin. Trying `http://10.10.10.138/admin/` fails, but `http://10.10.10.138/writeup/admin/` gets us a password prompt, so this software may well be installed.

Searchsploit turns up a bunch of potential exploits.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-15h34m49s120.png" class="kg-image"></figure>
# Initial Exploitation

The SQL injection that works on versions under 2.2.10 sounds like a good place to start as it doesn't require authentication. Reading the code it uses a timing vulnerability to extract username, email and password seed before trying to crack the password. Sounds perfect. I checked the script and verified that the URL it uses was present under /writeup as I could see a 200 status returned in the dev console. Lets run it.

    root@RD1734747W10:~/hackthebox/writeup# python 46635.py -u http://http://10.10.10.138/writeup/ --crack -w /usr/share/wordlists/rockyou.txt

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-16h04m24s353.png" class="kg-image"></figure>

I then tried the credentials on the admin url, nope, don't work. I then try them on ssh and we're in and can get the user flag.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-16h07m50s527.png" class="kg-image"></figure>
# Escalating privileges

Time for some more enumeration. I use LinPE at the point and the main thing it turns up are odd permissions on `/usr/local/bin` and `/usr/local/sbin` which allow accounts in the `staff` group to write files to the directories but not read them or list the directory contents.

Use LinPE, note odd permissions on /usr/local/bin and /usr/local/sbin. Nothing else turned up, so I tried [Pspy](https://github.com/DominicBreuker/pspy)next, which allows process monitoring without being root.

Running this turned up a cronjob running as root, but the permissions didn't let me get to this. I then tried starting a new ssh session while Pspy was running.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-16h13m14s512.png" class="kg-image"></figure>

Okay, this looks interesting. A file is run as root on login and the path means the odd `bin` and `sbin` directories will be evaluated first.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-16h17m30s539.png" class="kg-image"></figure>

The script just calls something called uname, so if we create a uname script in `/usr/local/bin` or `/usr/local/sbin` it will get called before the real uname.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-16h20m37s833.png" class="kg-image"></figure>

The classic Python reverse shell should work nicely, with a listener on Kali started with `nc -lvp 8080`. The `uname` script is copied to `/usr/local/sbin` You have to be fairly quick as a cleanup script runs that deletes these new created files. Start a new ssh session and:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-24-16h23m52s036.png" class="kg-image"></figure>