<div align="center">

# 🛡️ CTF Writeups — palantir1997

**VulnHub | Boot2Root | Penetration Testing Walkthroughs**

![GitHub last commit](https://img.shields.io/github/last-commit/palantir1997/CTF_Solve?color=red&style=flat-square)
![Writeups](https://img.shields.io/badge/Writeups-2-brightgreen?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-VulnHub-blue?style=flat-square)
![Lang](https://img.shields.io/badge/Language-KR%20%7C%20EN-orange?style=flat-square)

> 실전 모의침투 문제 풀이 기록 | Hands-on penetration testing writeups

</div>

---

## 📋 Writeup List

| # | Machine | Platform | Difficulty | Key Skills | Writeup |
|---|---------|----------|------------|------------|---------|
| 01 | 🚀 **Momentum: 1** | VulnHub | ⭐ Easy | Web Enum, Privilege Escalation | [📄 보기](./5_18_CTF_momentum1_writeup.md) |
| 02 | 🗡️ **Ripper: 1** | VulnHub | ⭐⭐ Medium | RIPS, Webmin, File Enumeration | [📄 보기](./5_20_CTF_ripper.md) |

---

## 🧰 주요 사용 기술 | Tech Stack

```
Reconnaissance     →  nmap, dirb, gobuster
Web Analysis       →  RIPS (PHP static analysis), robots.txt
Exploitation       →  SSH brute, credential stuffing
Privilege Escalation → find, /etc/passwd, log file analysis
```

## 🗺️ 공격 흐름 | General Attack Flow

```
[Network Scan]
      ↓
[Port & Service Enum]
      ↓
[Web Directory Bruteforce]
      ↓
[Vulnerability Discovery]
      ↓
[Initial Access (SSH)]
      ↓
[Local Enumeration]
      ↓
[Privilege Escalation]
      ↓
[Root Flag 🏁]
```

---

## 📂 풀이 목록 상세 | Writeup Details

### 🚀 01. Momentum: 1
- **플랫폼:** VulnHub
- **링크:** https://www.vulnhub.com/entry/momentum-1,685/
- **핵심 기술:** 웹 열거, 권한 상승
- **풀이:** [5_18_CTF_momentum1_writeup.md](./5_18_CTF_momentum1_writeup.md)

---

### 🗡️ 02. Ripper: 1
- **플랫폼:** VulnHub  
- **링크:** https://www.vulnhub.com/entry/ripper-1,706/
- **핵심 기술:** RIPS PHP 분석, Base64 디코딩, Webmin 로그 분석
- **풀이:** [5_20_CTF_ripper.md](./5_20_CTF_ripper.md)

---

### 🥔 03. Potato: 1
- **플랫폼:** VulnHub
- **링크:** https://www.vulnhub.com/entry/potato-1,529/
- **핵심 기술:** FTP 익명 로그인, PHP Type Juggling, LFI, MD5 해시 크랙, Sudo 경로 트래버설
- **풀이:** [5_15_CTF_potato.md](./5_15_CTF_potato.md)

---

## 🔍 Google 검색 키워드 | SEO Keywords

`vulnhub writeup` `CTF walkthrough` `boot2root` `ripper vulnhub` `momentum vulnhub`  
`privilege escalation` `RIPS PHP` `Webmin exploit` `CTF 풀이` `모의침투 실습`

---

<div align="center">

**⭐ 도움이 됐다면 Star 눌러주세요! | Star if this helped you!**

</div>
