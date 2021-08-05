---
layout: post
title: LaCasaDePapel Writeup
date: '2019-10-30 16:14:46'
tags:
- hack-the-box
---

## Enumeration

As usual, I started by running [Autorecon](https://github.com/Tib3rius/AutoRecon) and waiting for a bit. A quick look at the results showed 80, 443 and 21 were open on the box. Knowing there are some classic vulnerabilities with some FTP servers I had a look at port 21 first.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-18h25m42s990.png" class="kg-image"></figure>

Okay, so vsftpd is running and it’s version 2.3.4. This version has a [backdoor](https://www.hackingtutorials.org/metasploit-tutorials/exploiting-vsftpd-metasploitable/) that is often used as an example in training where you connect and send a username that includes :) which then opens a backdoor shell on port 6200.

At this point I have a look at the Nmap results for ports 80 & 443. Port 80 is running a fairly standard website that seems to have some signup for MFA enabled, and port 443 is requesting a client certificate for connections.

## Initial Foothold

### VSFTP & Psy Shell

I then give the backdoor a try and get the following:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-18h32m16s921.png" class="kg-image"></figure>

Okay, not the type of shell I was expecting, but that might be too easy. Psy Shell is a PHP debugging shell and allows you to enter PHP code rather than normal shell commands.

The first thing I try is a call out to systems commands by using a backtick, but this fails with `PHP Warning: shell exec() has been disabled for security reasons`.

At this point I messed around in PHP for a while. PHP isn’t a language I’ve spent a lot of time with, but with some googling I found the file and directory PHP functions which proved handy as I could use them for reading and writing files.

I had a good nose around the system and found I could write to `/home/dali/.ssh/authorized_keys` which meant I could my public key and login as this account via ssh, excellent. I went through the process for this, logged in and get the same Psy Shell! Doh, rabbit hole :(

I spent a bit more time messing in the file system and following some hints in the forum took a closer look at the PHP shell. I’d noticed earlier that a variable called `$tokyo` was declared in the shell when I ran `ls`. After using `help` I finally managed to dump it:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-18h47m20s936.png" class="kg-image"></figure>
### Client Certificate

Okay, this looks more like it. The function signs a certificate given a CSR and CA certificate and there is a pointer to where the CA key is stored. Lets try and grab it. (_I got distracted and spent 15 minutes nosing round the home directories at this point before getting back on track!_)

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-18h55m04s065.png" class="kg-image"></figure>

Okay, we can get the key. Obviously this should be secured away, so this could be handy. I also grab the certificate from port 443 on the web server by downloading it from the through the browser as I haven’t come across any other certificate and maybe I can use this as the CA cert.

Using Openssl I generate a signed certificate using the captured key and downloaded certificate:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-18h59m39s822.png" class="kg-image"></figure>

and then we need to change the format so we can import it in the browser

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h02m09s682.png" class="kg-image"></figure>

You don’t need to provide an export password, you can just hit enter. This can now be imported into the browser and we can try to browse to the secure website and see if the certificate will be accepted, which it wasn’t. At this point I spent an hour double-checking my OpenSSL commands, debugging the SSL connection using OpenSSL where I verified that the certificate should work as it was signed by the correct authority, and trying numerous browsers. I then took a break and came back later.

When I came back I noticed that the server had been reset and when I tried to connect to the secure site with the certificate, it worked fine! So, moral of the story, if you’re really sure what you’re doing is correct, a reset may be worth it as someone else may have broken the service.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h11m45s478.png" class="kg-image"></figure>
### Web Exploits

Now we’re in the secure area we seem to have some kind of episode download system.

Looking at how the path command is built for browsing the downloads, I try a local file inclusion

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h14m28s884.png" class="kg-image"></figure>

Yup, that works. Now I’ve already nosed around a lot using the PHP shell, but we are a different user, berlin, who I could access before. Messing around I get the idea of the directory structure

<figure class="kg-card kg-image-card kg-width-wide"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h16m57s832.png" class="kg-image"></figure>

I then noticed that the links for files contained some base64 encoding which when decoded were the path to the file. Having worked out the location of user.txt using the directory enumeration above I created a base64 string using [Cyberchef](https://gchq.github.io/CyberChef/)to try and download user.txt

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h21m32s220.png" class="kg-image"></figure><figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h22m44s005.png" class="kg-image"></figure>

Okay, part 1 done. This took 3:15:00, mainly due to faffing around with the certificates.

## Escalating Access

### Getting a full shell

Now we’re running as the `berlin` user I wonder if I can grab their ssh private key. I base64 encode the request and manage to download it. I then try to ssh in as `berlin` on the off chance they logged in locally, nope. From `/etc/passwd` I remember another user called `professor` so I tried to ssh in as that user and was in with a proper shell!

With a full shell I run `ls` and notice a`memcached.ini`and `memcached.js` file. Looking at `memcached.ini` we can see the contents look interesting

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h36m17s606.png" class="kg-image"></figure>

Now, it’d be good if we could edit this so we can get it run a command we want but the file is owned by root and we only have read permissions, hmm.

As this script feels like something that may be called from cron, I copy [PSpy](https://github.com/DominicBreuker/pspy)across to the box. Ideally we’d like root to be calling this script

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h40m37s549.png" class="kg-image"></figure>

Yup, UID 0 is running this, so anything we can put in the file will get run as root.

I create a new file called `memcached2.ini` with the following contents

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h44m26s297.png" class="kg-image"></figure>

and start a local listener with the usual `nc -lvp 1234` I then copy this file over the original file with `cp memcached2.ini memcached.ini` As the file is in a folder my current user owns I can overwrite/delete files in the folder even if I can’t write to them directly. We then wait for the listener to catch the shell

<figure class="kg-card kg-image-card"><img src="/content/images/2019/10/vlcsnap-2019-07-28-19h47m57s957.png" class="kg-image"></figure>

From user to root it took me another 01:23:00.

## Conclusion

This was a pretty interesting box, and it was the first time I’d played with client certificates. PHP isn’t really my thing, so this was a bit different and I found the client certificates not working very frustrating. Lots of other people seem to have had issues though, especially using Firefox.

I found root a lot easier, as this was much more back in my comfort zone.

