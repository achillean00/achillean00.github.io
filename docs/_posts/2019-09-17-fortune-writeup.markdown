---
layout: single
title: Fortune Writeup
date: '2019-09-17 16:12:59'
toc: true
header:
  overlay_image: /images/main-header.jpg
tags:
- hack-the-box
---

# Enumeration

As usual, `autorecon` was my go to. This turned up ssh on port 22, OpenBSD HTTPD on port 80 and an unknown service on 443. None of the services running were of a version with obvious vulnerabilities, so it was on to some manual enumeration.

The main website on port 80 seemed to providing a web front-end to the Unix fortune program, so that could be interesting. Connecting to 443 using a web browser didn't really seem to work.

I looked at fortune page first. I fired up Burpsuite and noticed that the when selecting a database for the fortune a POST contain `db=<dbname>` was sent to the server. I tried Local File Inclusion first, but this didn't work.

I then took a break and had a look at port 443. Using `gnutls-cli --insecure -d5 10.10.10.127` I could see a couple of certificates were sent, one for Bob and one for Charlie. Looking at the output it suggested the server wanted a client certificate, so finding a key to generate one would be handy.

# Exploitation

### Initial Foothold - Remote Code Execution

I went back to the fortune webapp and decided to try remote code execution, which is quick to test using the Burp repeater. `& sleep 2` didn't work, but `; sleep 2` did, so now we've a way to run commands. (01:22:00)

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-07-26-08h04m30s092.png" class="kg-image"></figure>

I tried to execute a reverse shell at this point, but various versions didn't work, so I stuck with the basic Remote Command Execution (RCE) via the webapp.

Using `cat` on `/etc/passwd` I could see there were two users called Bob and Charlie, which tallied with the certificates I saw earlier on port 443. I couldn't access `/home/charlie` but could access `/home/bob`. There was also a user called nfsuser, which had an odd shell, authpf.

Bob had some interesting directories called `ca` and `dba`. The `ca` directory seemed to be for creating a certificate authority. The `dba` directory had a SQL statement for setting up a table to hold authorized\_keys, suggesting some interaction with ssh.

I grabbed the certificates I could find `/home/bob`. Most interestingly under the sub-directory `intermediate` the `private` directory was readable and I was able to get the intermediate key, and certificate.

At this point I spent a good hour going down rabbit holes due to how the ssh &nbsp;authorized\_keys were stored in a postgres database, before deciding to try &nbsp;converting the intermediate certificate and key to PKCS12 format to see if would be accepted by the service on port 443:

    openssl pkcs12 -export -in intermediate.pem -inkey intermediate.key -out inter.pkcs12

I then imported this into my personal store on Chromium and tried to go to `https://10.10.10.127`:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-22h23m33s597.png" class="kg-image"></figure>

Okay, looking good. Next we click _generate_ to an private key and save this locally. We can then try to ssh to the box. I use the `nfsuser` user as looking at `/etc/passwd` and `/etc/ssh/sshd_config` I can see this account is a bit special and keys handled in a different way.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-22h26m47s049.png" class="kg-image"></figure>
### Increasing Access - NFS

So, after fixing the permissions on the private key file we are in. However, it's a captive shell using [authpf](https://www.openbsd.org/faq/pf/authpf.html). This is usually used as a gateway onto a private network, so at this point we'll do some more enumeration to see if we can see anything new.

I run autorecon again and we can now see port 2049 (nfs) is visible and that `/home` is exported to `everyone`. Handy.

I mount this using:

    mkdir mount
    mount -t nfs 10.10.10.127/home /root/hackthebox/fortune/mount

At this point I still can't get into the `charlie` folder the permissions only allow the user and group access. However, the NFS export has been made without the all\_squash enabled which would map the _uids_ to a unpriviledged account. As that hasn't happened I can make an account with the same _uid_ as the account I want to access (1000 for charlie) and then because I'm root on my local Kali box I can su to that account and linux will let me in.

    useradd -u 1000 charlie
    passwd charlie
    su charlie

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-22h43m27s406.png" class="kg-image"></figure>

And we have user.txt. This took a while, around 04:50:00.

### Full shell access

Given I have full access to `/home/charlie` I now put my public key in `/home/charlie/.ssh/authorized_keys` and ssh in as _charlie_. A full shell at last!

I do an `ls -al` in his home directory out of habit and find he has an _mbox_ file which may contain mail.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-22h48m21s953.png" class="kg-image"></figure>
### Analysing pgadmin4

This suggests the dba password in pgadmin4 is our target. I find this in `/usr/local` and the version is 4-3.4, which is important. Reading the documentation they improved the way cached passwords were encrypted in later versions to use a master password, but in this version they are encrypted using the key for the user entering the data. Note: this isn't the plaintext password of the user, but the encrypted key material that is stored in the database.

Looking at the config we can see the config database for the application is stored at `/var/appsrv/pgadmin4/pgadmin4.db` and that it's in SQLite format. Kali has a browser for this, so we'll copy the file to our Kali box and have a nose.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-22h57m05s007.png" class="kg-image"></figure>

This is interesting. Here is the encrypted dba password and it's associated with user\_id 2.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-22h59m41s518.png" class="kg-image"></figure>

User 2 is _bob_ and there is his encrypted password data.

I spent a bunch of time hunting through the code for the correct encryption/decryption routines at this point, but failed to notice I'd downloaded a newer version of the code! I went back to hunt through the correct version. The screenshot below shows the code which indicates the connection password is encrypted with the logged in users _user.password_, which from the code is the encrypted string, as mentioned before.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-23h08m46s632.png" class="kg-image"></figure>

Hunting around I find a python file that has the encryption and decryption routines called `crypto.py` (duh!).

Having a read, I re-purpose this to decrypt the _dba_ password.

    import base64
    import hashlib
    
    from Crypto import Random
    from Crypto.Cipher import AES
    
    padding_string = b'}'
    
    root = 'utUU0jkamCZDmqFLOrAuPjFxL0zp8zWzISe5MF0GY/l8Silrmu3caqrtjaVjLQlvFFEgESGz'
    user = '$pbkdf2-sha512$25000$z9nbm1Oq9Z5TytkbQ8h5Dw$Vtx9YWQsgwdXpBnsa8BtO5kLOdQGflIZOQysAy7JdTVcRbv/6csQHAJCAIJT9rLFBawClFyMKnqKNL5t3Le9vg'
    
    def decrypt(ciphertext, key):
        """
        Decrypt the AES encrypted string.
    
        Parameters:
            ciphertext -- Encrypted string with AES method.
            key -- key to decrypt the encrypted string.
        """
    
        global padding_string
    
        ciphertext = base64.b64decode(ciphertext)
        iv = ciphertext[:AES.block_size]
        cipher = AES.new(pad(key), AES.MODE_CFB, iv)
        decrypted = cipher.decrypt(ciphertext[AES.block_size:])
    
        return decrypted
    def pad(key):
        """Add padding to the key."""
    
        global padding_string
        str_len = len(key)
    
        # Key must be maximum 32 bytes long, so take first 32 bytes
        if str_len > 32:
            return key[:32]
    
        # If key size id 16, 24 or 32 bytes then padding not require
        if str_len == 16 or str_len == 24 or str_len == 32:
            return key
    
        # Convert bytes to string (python3)
    
        if not hasattr(str, 'decode'):
            padding_string = padding_string.decode()
    
        # Add padding to make key 32 bytes long
        return key + ((32 - str_len % 32) * padding_string)
    
    output = decrypt (root,user)
    print (output)

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-23h14m21s393.png" class="kg-image"></figure>

Now we have the root password, we can `su` from our ssh session as _charlie_ and grab the root flag. This took another 2 hours from getting the user flag.

<figure class="kg-card kg-image-card"><img src="/content/images/2019/09/vlcsnap-2019-08-02-23h16m00s960.png" class="kg-image"></figure>
# Conclusion

This was my first Insane level box. Some bits were fairly tricky, mainly due to my inexperience in the area, but overall it didn't feel as bad as some of the easy boxes that I've worked through. I still have a habit of going off on a tangent though, which wastes time!

