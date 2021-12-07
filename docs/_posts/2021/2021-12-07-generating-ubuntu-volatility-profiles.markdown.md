---
layout: single
title: Generating Ubuntu Volatility profiles
date: '2021-08-06 19:42:26'
tags:
- dfir
classes: wide
typora-copy-images-to: ../images/posts/2021/${filename}/
---

This post is mainly for my own reference as I couldn't really find a clear guide for all the steps.

## Scenario

I recently needed to do some analysis of an Ubuntu machine. I didn't really want to be running any more commands than I really needed to on the machine, creating profiles on a possibly compromised host feels a little off, so I wanted to do this on a clean machine.

## Tools used

- [F-Secure Cat-Scale](https://github.com/FSecureLABS/LinuxCatScale) to grab the IR triage information
- [Volatility2](https://github.com/volatilityfoundation/volatility) to analyse the memory image

## Kernel Information

In order to build a profile you need to have the same kernel version running as the host. On a live system you can find this by running `uname -r` but without interactive access we can rely on Cat-Scale which will collect the release information which reveals the following

```bash
VERSION="18.04.1 LTS (Bionic Beaver)"
KERNELVERSION(uname -r)=4.15.0-132-generic
```

## VM and Kernel installation

To start with we grab a copy of the latest 18.04 release from Ubuntu and install this as normal in a virtual machine. It's pretty likely that this will be running a later version of the kernel, which we can confirm with `uname -r`

```bash
4.15.0-163-generic
```

Thankfully installing a new kernel is pretty easy as we know exactly what version we want

```bash
apt install linux-headers-4.15.0-132-generic
```

Once installed we need to reboot the VM to switch to the old kernel. When the VM is starting you need to *hold down the shift key* to trigger the Grub menu, choose the *Advanced* options and then choose the kernel version needed.

Once logged back into the VM a `uname -r` should show you're now running the kernel version needed.

## Generating the profile

Firstly install the build-essential package

`apt install build-essential`

Then we follow the excellent instructions outlined [here](https://www.andreafortuna.org/2019/08/22/how-to-generate-a-volatility-profile-for-a-linux-system/):

### Clone the volatility repository

```bash
git clone https://github.com/volatilityfoundation/volatility.git
```

### Create the module.dwarf

```bash
cd volatility/tools/linux 
make 
```

### Create a zip file of module.dwarf and the debug symbols in System.map

```bash
zip $(lsb_release -i -s)_$(uname -r)_profile.zip ./volatility/tools/linux/module.dwarf /boot/System.map-$(uname -r)
```

You can now copy this zip to your forensic workstation with volatility installed and put it in `volatility/volatility/plugins/overlays/linux`.

When you run `python vol.py --info` you should now see the new profile listed

