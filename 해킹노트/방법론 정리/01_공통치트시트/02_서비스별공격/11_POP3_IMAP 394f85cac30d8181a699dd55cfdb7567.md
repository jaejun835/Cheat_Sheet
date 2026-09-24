# 11_POP3_IMAP

# POP3 / IMAP (110 / 143 / 993 / 995)

메일 수신 프로토콜. SMTP와 별개로 사용자 열거가 가능한 경우가 있고, 평문 인증 거부 여부가 핵심 확인 포인트다.

---

## 포트 구분

| 포트 | 프로토콜 | 암호화 |
| --- | --- | --- |
| 110 | POP3 | 평문 |
| 143 | IMAP | 평문 |
| 993 | IMAPS | SSL/TLS |
| 995 | POP3S | SSL/TLS |

---

## 1. 열거

```bash
nmap -sV -p 110,143,993,995 --script=pop3-capabilities,imap-capabilities <IP>
# 배너/지원 기능(인증 방식 포함) 확인

nc -nv <IP> 110
telnet <IP> 110
# +OK Dovecot (Ubuntu) ready. 같은 배너로 서비스·버전 확인
```

---

## 2. POP3 USER 명령으로 사용자 열거

```bash
telnet <IP> 110
USER root
# +OK 응답 오면 계정 존재
# -ERR 응답이면 계정 없음 (또는 인증 방식 자체가 거부하는 설정일 수도 있음)

USER www-data
# -ERR [AUTH] Plaintext authentication disallowed on non-secure (SSL/TLS) connections.
# → 평문 인증 자체를 서버가 거부하는 경우, 이 메시지는 계정 존재여부와 무관함 — 오판단 주의

# 자동화
hydra -L users.txt -p x pop3://<IP>
# 유효한 계정을 비밀번호 하나로 대량 대입해서 응답 차이로 간접 열거하는 방식 (직접적인 USER 열거 도구가 따로 없을 때 대체)
```

---

## 3. 인증 방식 확인 (평문 허용 여부)

```bash
telnet <IP> 143
# a1 CAPABILITY
# * CAPABILITY IMAP4rev1 ... AUTH=PLAIN AUTH=LOGIN STARTTLS ...
# STARTTLS 없이 AUTH=PLAIN/LOGIN만 있으면 평문 인증이라 패탷 캐프처로 크리덴셔 탈취 가능합 있음 (내부망/MITM 가능 상황에서)
```

---

## 4. 로그인 시도 (크리덴셔 확보했을 때)

```bash
# POP3
telnet <IP> 110
USER <사용자>
PASS <비밀번호>
# +OK Logged in. 이면 성공, 메일 박스 접근 가능
LIST
# 메일 목록 번호/크기 확인
RETR 1
# 1번 메일 전체 내용 가져오기 (크리덴션, 내부 링크 등 정보 획득 가능)

# IMAP (curl이 훨씬 편함)
curl -k 'imap://<IP>' --user <사용자>:<비밀번호>
curl -k 'imaps://<IP>/INBOX' --user <사용자>:<비밀번호>
```

---

## 팁

- SMTP에서 VRFY로 찾은 사용자 리스트를 POP3/IMAP 계정 확인용으로 재활용 가능
- SSL/TLS 강제(993/995)로만 운영하는 서버는 USER 명령 자체가 거부될 수 있음 — 이때는 계정 존재를 다른 서비스(SMTP VRFY, SSH 등)로 크로스링해야 함
- 크리덴셔 확보 후에는 메일 박스 자체가 정보 수집용 타겟이 될 수 있음 (내부망 IP, 링크, 다른 계정 힌트 등)