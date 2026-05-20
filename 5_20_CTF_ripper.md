# 🗡️ Ripper: 1 — CTF Writeup

<img width="384" height="492" alt="Image" src="https://github.com/user-attachments/assets/a68f6ad1-8be5-4657-a9f0-e69aeaddf731" />


> **Platform:** VulnHub | **URL:** https://www.vulnhub.com/entry/ripper-1,706/  
> **Difficulty:** Intermediate | **Goal:** `/root/flag.txt` 획득

---

## 📡 1단계: 네트워크 스캔 — 타겟 IP 찾기

```bash
nmap -sn 172.16.11.0/24
```

<img width="817" height="330" alt="Image" src="https://github.com/user-attachments/assets/13f0d3d0-246b-4445-9ef0-0b02ecee97f5" />

> 네트워크 대역에서 살아있는 호스트를 핑 스캔으로 탐색.  
> → 타겟 IP: `172.16.11.227` 발견

---

## 🔍 2단계: 포트 & 서비스 스캔

```bash
nmap -p- -sV 172.16.11.227
```

> 전체 포트(0~65535) 대상으로 서비스 버전 스캔.  
> → 포트 **80** (HTTP), **22** (SSH), **10000** (Webmin) 등 확인

<img width="912" height="825" alt="Image" src="https://github.com/user-attachments/assets/505abee8-18db-450f-bed0-734cd290268b" />

---

## 🕷️ 3단계: 웹 디렉토리 브루트포싱

```bash
dirb http://172.16.11.227/
```

> 기본 wordlist로 숨겨진 디렉토리 탐색.

---

## 🌐 4단계: 도메인 등록 & Webmin 접속

```bash
sudo vi /etc/hosts
```

아래 줄 추가:

```
172.16.11.227   ripper-min
```

이후 브라우저에서 접속:

```
https://ripper-min:10000
```

> `/etc/hosts`에 도메인을 등록해 가상 호스트 기반 라우팅 우회.


<img width="914" height="827" alt="Image" src="https://github.com/user-attachments/assets/dd2c5945-9f4a-4497-9eb7-67dc34f216a9" />

---

## 🔎 5단계: Gobuster 디렉토리 열거

```bash
sudo gobuster dir \
  -u http://ripper-min \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,log,bak \
  -t 80 \
  --no-error
```

> php, txt, log, bak 확장자까지 커버하는 정밀 탐색.

---

## 🤖 6단계: robots.txt & Base64 힌트 디코딩

```
https://ripper-min:10000/robots.txt
```


<img width="905" height="795" alt="Image" src="https://github.com/user-attachments/assets/30ea843a-9c1b-4c78-83ea-b2ef4f7b6edf" />

robots.txt에서 발견한 Base64 문자열 디코딩:

```bash
echo d2Ugc2NhbiBwaHAgY29kZXMgd2l0aCByaXBzCg== | base64 -d
```

**결과:**
```
we scan php codes with rips
```


<img width="905" height="645" alt="Image" src="https://github.com/user-attachments/assets/fc1f1558-f97e-47a5-862e-8837c816cf10" />

<img width="909" height="827" alt="Image" src="https://github.com/user-attachments/assets/67cad39c-e003-4ba8-8096-6891d8a35b5e" />

> 🔑 **핵심 힌트:** `rips` — PHP 소스코드 분석 도구  
> → 80번 포트로 `http://ripper-min/rips` 접속

---

## 🧪 7단계: RIPS로 PHP 소스코드 분석

브라우저에서 접속:

```
http://ripper-min/rips
```

<img width="917" height="867" alt="Image" src="https://github.com/user-attachments/assets/a9887efa-a874-4ee2-97e7-8b89a5ab609b" />


분석 경로 입력:

```
/var/www/html/rips
```


<img width="1163" height="929" alt="Image" src="https://github.com/user-attachments/assets/2047d8be-fcf8-4733-aafa-b9930dd984a0" />

<img width="1168" height="926" alt="Image" src="https://github.com/user-attachments/assets/f8c30976-9d69-4caf-9e25-43e9d06373ac" />

> RIPS가 PHP 소스코드를 정적 분석해 취약점 및 숨겨진 파일 노출.  
> → `secret.php` 발견 → **아이디/비밀번호** 획득

<img width="650" height="360" alt="Image" src="https://github.com/user-attachments/assets/2d4bf062-9b33-4a9b-b0f3-4460b6c401ce" />

---

## 🔐 8단계: SSH 접속 & 첫 번째 플래그

```bash
ssh ripper@172.16.11.227
# secret.php에서 획득한 비밀번호 사용
```

<img width="595" height="216" alt="Image" src="https://github.com/user-attachments/assets/fd4f93fa-7219-426d-946e-74158e3d5826" />

```bash
cat flag.txt
```

> 🎉 첫 번째 플래그 획득!

---

## 👥 9단계: 로컬 사용자 열거 — 다음 타겟 찾기

```bash
cat /etc/passwd | grep bash
```

<img width="532" height="92" alt="Image" src="https://github.com/user-attachments/assets/7675d718-b7d5-4238-aab7-02345d0f7c1b" />

> **목적:** bash 쉘을 가진 실제 로그인 가능 사용자만 필터링.  
> ripper 계정으론 `/root`에 접근 불가 → 권한 상승을 위한 다른 계정 탐색.  
> → `cubes` 계정 발견


```bash
find / -user cubes -type f -exec ls -al {} \; 2>/dev/null
```

<img width="800" height="186" alt="Image" src="https://github.com/user-attachments/assets/855cdbb1-e1ec-4ae6-b6ef-ba6c43a809a6" />

<img width="809" height="140" alt="Image" src="https://github.com/user-attachments/assets/c776fcfb-c010-4f54-8c40-a445c0caf4b9" />

> cubes / I100tpeople ssh접속

---

## 🔎 10단계: cubes 소유 파일 전체 탐색

```bash
find / -user cubes -type f -exec ls -al {} \; 2>/dev/null
```

<img width="801" height="456" alt="Image" src="https://github.com/user-attachments/assets/cb58297a-14a4-4960-9d0f-3d037013fcde" />

> **목적:** 서버 전체에서 `cubes`가 소유한 파일을 샅샅이 탐색.  
> 홈 디렉토리 외 숨겨진 파일에서 크리덴셜 힌트를 낚는 것이 목표.  
>
> → **발견:** `/var/webmin/backup/miniser.log`

```bash
cat /var/webmin/backup/miniser.log
```

<img width="945" height="463" alt="Image" src="https://github.com/user-attachments/assets/0077f970-51a8-42c2-850e-aa3313ca919f" />

> → 크리덴셜 발견: `admin / tokiohotel`

---

## 🔄 11단계: cubes 계정 전환

```bash
su cubes
# 비밀번호: tokiohotel
```

---

## 🏁 12단계: 루트 플래그 획득

```bash
cd /root
ls -al
cat flag.txt
```

> 🎊 **루트 플래그 획득 완료!**


<img width="384" height="492" alt="Image" src="https://github.com/user-attachments/assets/a68f6ad1-8be5-4657-a9f0-e69aeaddf731" />


## 🗺️ 공격 흐름 요약

```
네트워크 스캔
    ↓
포트/서비스 열거
    ↓
웹 디렉토리 브루트포싱
    ↓
/etc/hosts 등록 → Webmin(10000) 접속
    ↓
robots.txt → Base64 디코딩 → RIPS 힌트 획득
    ↓
RIPS로 secret.php 발견 → SSH 크리덴셜 확보
    ↓
ripper 계정 SSH 접속 → flag.txt #1
    ↓
/etc/passwd 분석 → cubes 계정 발견
    ↓
find로 cubes 소유 파일 탐색 → miniser.log → admin/tokiohotel
    ↓
su cubes → Webmin admin 로그인
    ↓
/root/flag.txt 획득 🏁
```

---

## 🛠️ 사용 도구

| 도구 | 용도 |
|------|------|
| `nmap` | 네트워크/포트 스캔 |
| `dirb` | 웹 디렉토리 열거 |
| `gobuster` | 확장자 포함 정밀 디렉토리 열거 |
| `RIPS` | PHP 소스코드 정적 분석 |
| `base64` | 인코딩 힌트 디코딩 |
| `find` | 파일 시스템 권한 탐색 |
