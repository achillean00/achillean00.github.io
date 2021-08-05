---
layout: post
title: Beep Writeup
date: '2019-12-18 17:00:00'
tags:
- hack-the-box
---

## Enumeration

Starting with Autorecon a huge slew of ports was found, which can sometimes make it tricky to identify the vulnerable services.

As 80 & 443 were open I decided to take a look at those first and found that Elastix was running, which seems to be a VOIP solution based on Asterix. A quick look at Exploitdb suggested that some version were [vulnerable](https://www.exploit-db.com/exploits/37637) to local file inclusion, which is quick and easy to test.

## Exploitation

Looking at the vulnerability details I tried visiting

    https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action

and verified I could indeed see the config file for this service.

The config also included a number of passwords:

<figure class="kg-card kg-image-card"><img src="/content/images/2019/12/Screenshot-from-2019-12-18-10-00-44.png" class="kg-image"></figure>

These allowed you to login to Elastix as an admin, but I thought I'd see if they'd been reused elsewhere. I tried the user account (_fanis_) I'd found earlier by looking at /etc/password for the LFI but this didn't work via ssh.I then tried the password on the root account via ssh and was in. It was then trivial to get the two flags.

