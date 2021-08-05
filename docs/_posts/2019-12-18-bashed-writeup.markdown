---
layout: post
title: Bashed Writeup
date: '2019-12-18 17:05:00'
tags:
- hack-the-box
---

## Enumeration

The usual Autorecon run just turned up a webserver running on Port 80. Having a look at this in Firefox showed that it seemed to be a site about a PHPBash application that the site owner had developed on this very server!

Nikto had highlighted a /dev directory existed and going there gave us access to a webshell.

## Exploitation

Running the PHPBash shell we could quickly grab the user flag:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/12/Screenshot-from-2019-12-18-10-16-30.png" class="kg-image"></figure>

We couldn't grab the root flag though as the shell was running as the www-data user which didn't have access.

## Privilege Escalation

Running `sudo -l` I could see that the www-data user could run any command as the `scriptmanager` user.

Looking around the filesystem I could see a directory `/scripts` owned by `scriptmanager`. Using `sudo -u scriptmanager ls -al /scripts` I could see the directory contained a python script, `test.py` which just created a file called `test.txt`. However, the `test.txt` file was owned by root, suggested a cronjob was running as root that was calling the Python script.

At this point I could have changed the Python script just to get me the contents of `/root/root.txt` but it seemed more fun to get a full shell.

I created a local python file containing the reverse shell from [Pentestmonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) and ran the Python SimpleHTTPServer on my Kali box so I could pull it across to the target using `wget`. I started a listener on my Kali box with `nc -lvp 1234` and copied the shell script across from the temporary diorectory I'd stored it in to `/scripts/test.py` and waited a short while.

As expected, the script was executed by the root cronjob and I had a root shell and I quickly grabbed the root flag `cc4f0afe3a1026d402ba10329674a8e2`

