---
layout: single
title: Sharing a Zebra GK420D via CUPS and making it work on MacOS and Windows
date: '2025-12-05 18:30:47'
tags:
- "tech support"
classes: wide
typora-copy-images-to: ../images/posts/2025/${filename}/
---

## Introduction

I've had the Zebra GK420d for a while, which I purchased second hand off eBay. Apart from needing a new print head it's been great, but the whole family keeps sending me label to print and I figured it was time to make it available over the network.

The CUPS side of this was pretty straightforward, but getting it to work on MacOS and Windows was a bit of a pain, so I'm documenting it, mainly so I don't forget but it may be of use to others too!



## CUPS

I used a [Pi Zero W](https://thepihut.com/products/raspberry-pi-zero-w) with Raspberry Pi OS (32-bit) installed, I'm not going into details on the basic install as it's well documented.

I installed cups

```
apt install cups
```

Added my local user to the lpadmin group

```
sudo usermod -aG lpadmin me
```

Tweaked `/etc/cupsd.conf` to open up the web interface and allow admin access

```
# Listen on all interfaces
Port 631

# Restrict access to the server...
<Location />
  Order allow,deny
  Allow from 192.168.1.*
  Allow localhost
</Location>

# Restrict access to the admin pages...
<Location /admin>
  AuthType Default
  Require valid-user
  Allow all
  Order allow,deny
</Location>

# Restrict access to configuration files...
<Location /admin/conf>
  AuthType Default
  Require valid-user
  Allow all
  Order allow,deny
</Location>

# Restrict access to log files...
<Location /admin/log>
  AuthType Default
  Require valid-user
  Allow all
  Order allow,deny
</Location>
```

The changes basically open up the web interface to the local network restrict the webpage to one network range and allow local users to auth to the admin settings.

This isn't a very secure setup but is fine for my home network.

I connected the printer for USB and then used the admin interface running on port 631 to add the printer by following the wizard. When asked the make, select **Raw**

I originally use the Zebra ZPL driver but found that either jobs wouldn't print or would end up being scaled all the way in, which seemed to be due to the print driver in CUPS and the one in the client OS fighting each other.

At this point CUPS is ready to go.

## MacOS

MacOS comes with a suitable driver installed, so this saves a step.

1. Fire up Printers and Scanners and click **Add Printer**
2. The Zebra should appear under default as a Bonjour printer
3. Click on it and under **Use** choose **Select Software** and choose **Zebra ZPL Label Printer**
4. Okay the dialogues and the printer will be added. Try printer a label.

I did originally just use the Airprint version that will be detected in a print dialog, but I had the scaled in problems when using that. This method uses the same driver as you'd use if it was connectly directly via USB and seems to work fine.

When printing you will need to select the correct label size, orientation etc. but you can save this as a preset.

## Windows

As usual, Windows was slightly more of a pain.

Before doing anything else download the printer drivers from https://www.zebra.com/gb/en/support-downloads/printers/printer-drivers.html

The v5 drivers were the ones I used as the GK420d is old now and doesn't seem to be supported by the later drivers.

Once the drivers are installed it's time to add the printer.

### Add the printer

1. Navigate to the web interface of your CUPS install and select the printer. Copy the Url, it will be something like `https://192.168.2.87:631/printers/Zebra_Technologies_ZTC_GK420d`. This will be used to add the printer, but you must change the url to **http**, https won't work!
2. Open up Printers and Scanners and click Add device at the top
3. Wait a bit and then click Add a new device manually
4. In the dialog choose Select a shared printer by name and paste in the url `http://192.168.2.87:631/printers/Zebra_Technologies_ZTC_GK420d` and click next
5. In the driver dialog the Manufacturer is Zdesign and the Printer is ZDesigner GK420d. Click OK
6. It should be installed successfully, click Next and then Finish



### Configuration

1. Back in the Printers and scanner window click your newly added device

2. Choose printing preferences

   1. Under options set the paper format to inch
   2. Under stocks define a New stock with the details for your labels. I use

   | Options      | Value  |
   | ------------ | ------ |
   | Darkness     | 30     |
   | Speed        | 4      |
   | Label Width  | 4 inch |
   | Label Height | 6 inch |

3. Back in options, set your stock to the one you just defined.

4. OK the dialogs

The printer should now be ready to go.

I found that adding the printer over IPP, which is the default if you just add the printer that Windows wil autodiscover doesn't work, even if you specify the correct driver. For me the queue just errored all the time, whilst this method seems to work fine.

