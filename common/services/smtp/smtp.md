# SMTP

# SMTP (25 / 587 / 465)

이메일 전송 프로토콜. 사용자 열거, 릴레이 악용, 인증 브루트포스가 주요 공격 벡터.

---

## 1. 열거

```bash
# nmap
nmap -p 25,587,465 --script=smtp-commands,smtp-enum-users,smtp-vuln-cve2010-4344 <IP>
nmap -sV -p 25 <IP>

# 배너 확인
nc -nv <IP> 25
telnet <IP> 25
```

---

## 2. 사용자 열거

### VRFY 명령 (사용자 존재 확인)

```bash
nc -nv <IP> 25
# VRFY root
# 252 2.0.0 root        ← 존재함
# 550 5.1.1 root... User unknown  ← 존재하지 않음

# 또는
telnet <IP> 25
VRFY admin
VRFY john
```

### EXPN 명령 (메일링 리스트 확장)

```bash
EXPN postmaster
EXPN root
```

### RCPT TO 방식 (VRFY 막혀있을 때)

```bash
nc -nv <IP> 25
HELO attacker.com
MAIL FROM: <test@test.com>
RCPT TO: <root@target.com>      # 250 OK → 존재 / 550 → 없음
RCPT TO: <admin@target.com>
```

### smtp-user-enum (자동화)

```bash
smtp-user-enum -M VRFY -U /usr/share/wordlists/metasploit/unix_users.txt -t <IP>
smtp-user-enum -M RCPT -U users.txt -D target.com -t <IP>
smtp-user-enum -M EXPN -U users.txt -t <IP>
# -v 옵션 추가시 실행 상태 확인 가능 

# nmap
nmap -p 25 --script=smtp-enum-users --script-args smtp-enum-users.methods=VRFY <IP>
```

---

## 3. 오픈 릴레이 확인

```bash
# nmap 자동 스크립트 (먼저 이거부터 시도)
nmap -p25 -Pn --script smtp-open-relay <IP>
# "Server doesn't seem to be an open relay, all tests failed" 이면 안전
# 취약하면 허용된 릴레이 테스트 목록을 구체적으로 보여줌

# 수동 확인
nc -nv <IP> 25
HELO test
MAIL FROM: <fake@external.com>
RCPT TO: <victim@otherdomain.com>    # 250 OK → 오픈 릴레이 취약
DATA
Subject: Test
This is a test.
.
QUIT
```

---

## 4. 인증 브루트포스

```bash
# hydra
hydra -l admin -P /usr/share/wordlists/rockyou.txt smtp://<IP>
hydra -l admin -P rockyou.txt -s 587 smtp://<IP>

# 인증 방식 확인 (AUTH 목록)
nc -nv <IP> 25
EHLO test
# 250-AUTH LOGIN PLAIN CRAM-MD5 ← 지원 인증 방식
```

---

## 5. 이메일 전송 (정보 수집 후)

```bash
# sendmail / swaks
swaks --to admin@target.com --from test@test.com \
  --server <IP> --body "Test email"

# python
python3 -c "
import smtplib
s = smtplib.SMTP('<IP>', 25)
s.sendmail('from@test.com', 'admin@target.com', 'Subject: Test\n\nBody')
s.quit()
"
```

---

## 팁

- VRFY 막혀있으면 RCPT TO 방식 시도
- 오픈 릴레이 → 스팸/피싱 전송 가능
- 25 포트 (서버 간), 587 포트 (클라이언트 인증), 465 (SSL)
- `EHLO` 명령으로 서버 지원 기능 확인