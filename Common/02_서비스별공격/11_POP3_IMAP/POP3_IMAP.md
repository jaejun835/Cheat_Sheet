# 11_POP3_IMAP

# POP3 / IMAP (110 / 143 / 993 / 995)

메일 수신 프로토콜. SMTP 와 별개로 사용자 열거가 되는 경우가 있고, 평문 인증 허용 여부가 핵심 확인 포인트다. 자격증명을 확보하면 메일함 자체가 내부 정보(크리덴셜, 내부 링크, 다른 계정 힌트)의 보고가 된다. IMAP 은 서버에 메일을 남겨두므로 POP3 보다 뒤질 게 많다.

---

## 포트 구분

| 포트 | 프로토콜 | 암호화 |
| --- | --- | --- |
| 110 | POP3 | 평문 |
| 143 | IMAP | 평문 |
| 993 | IMAPS | SSL/TLS |
| 995 | POP3S | SSL/TLS |

평문 포트(110/143)는 telnet/nc 로, TLS 포트(993/995)는 openssl 로 붙는다. telnet 으로 993/995 에 붙으면 암호화 핸드셰이크가 안 돼서 깨진다.

---

## 1. 열거

```bash
# nmap - 배너, 지원 기능(인증 방식 포함), 알려진 취약점
nmap -sV -p 110,143,993,995 <IP>
nmap -p 110,143,993,995 --script=pop3-capabilities,imap-capabilities <IP>
nmap -p 110 --script=pop3-ntlm-info <IP>     # NTLM 정보로 도메인/호스트명 노출되기도 함

# 배너 직접 확인
nc -nv <IP> 110
telnet <IP> 110
# +OK Dovecot (Ubuntu) ready. 같은 배너로 서비스/버전 확인
```

---

## 2. 사용자 열거

POP3 의 USER 명령은 계정 존재 여부에 따라 응답이 갈리는 경우가 있다. 다만 평문 인증을 막아둔 서버는 이 응답이 계정 유무와 무관하니 오판에 주의한다.

```bash
telnet <IP> 110
USER root
# +OK            → 계정 존재
# -ERR           → 계정 없음 (또는 서버가 평문 인증 자체를 거부하는 설정)

USER www-data
# -ERR [AUTH] Plaintext authentication disallowed on non-secure (SSL/TLS) connections.
#   → 이 메시지는 계정 존재 여부와 무관함. 이럴 땐 TLS 포트(995)로 붙어서 확인
```

```bash
# 자동 열거 - 비밀번호 하나로 다수 계정을 대입해 응답 차이로 간접 열거
hydra -L users.txt -p whatever pop3://<IP>

# SMTP 에서 VRFY/RCPT 로 찾은 사용자 목록을 여기 그대로 재활용
```

---

## 3. 인증 방식 확인 (평문 허용 여부)

STARTTLS 없이 AUTH=PLAIN/LOGIN 만 있으면 평문 인증이다. 내부망이나 MITM 상황이면 패킷 캡처로 자격증명을 탈취할 수 있다.

```bash
# IMAP capability
telnet <IP> 143
a1 CAPABILITY
# * CAPABILITY IMAP4rev1 ... AUTH=PLAIN AUTH=LOGIN STARTTLS ...
#   STARTTLS 가 없고 AUTH=PLAIN/LOGIN 만 있으면 평문 전송

# POP3 capability
telnet <IP> 110
CAPA
# STLS 가 있으면 STARTTLS 지원, 없으면 평문
```

---

## 4. 브루트포스

평문 포트는 서비스명 그대로, TLS 포트는 s 붙은 서비스명(pop3s/imaps)으로 때린다.

```bash
# POP3 / IMAP (평문)
hydra -L users.txt -P passwords.txt pop3://<IP>
hydra -L users.txt -P passwords.txt imap://<IP>

# POP3S / IMAPS (TLS) - 서비스명에 s, 또는 -S 플래그 + 포트 지정
hydra -L users.txt -P passwords.txt pop3s://<IP>
hydra -L users.txt -P passwords.txt imaps://<IP>
hydra -L users.txt -P passwords.txt -s 995 -S pop3://<IP>

# 단일 계정 비번 스프레이 (잠금 회피)
hydra -l <user> -P passwords.txt pop3://<IP> -t 4
```

---

## 5. 인증 후 메일함 접근

### POP3 (110)

```bash
telnet <IP> 110
USER <user>
PASS <password>
# +OK Logged in.
STAT            # 메일 개수와 총 크기
LIST            # 메일별 번호와 크기
RETR 1          # 1번 메일 전체 가져오기
TOP 1 20        # 1번 메일 헤더 + 본문 앞 20줄만
DELE 1          # 1번 삭제 (주의)
QUIT
```

### IMAP (143)

IMAP 은 모든 명령 앞에 태그(a1, a2 …)를 붙인다. 폴더 구조가 있어 POP3 보다 뒤질 게 많다.

```bash
telnet <IP> 143
a1 LOGIN <user> <password>
a2 LIST "" "*"            # 전체 폴더(메일박스) 나열
a3 SELECT INBOX           # INBOX 열기 (메일 개수 표시됨)
a4 FETCH 1 BODY[]         # 1번 메일 전체(헤더+본문)
a5 FETCH 1:* (FLAGS)      # 전체 메일 플래그 목록
a6 SEARCH ALL             # 전체 메일 번호
a7 SEARCH BODY "password" # 본문에 password 포함된 메일 검색
a8 SEARCH FROM "admin"    # 특정 발신자 검색
a9 LOGOUT
```

### curl (가장 편함)

```bash
# 폴더 목록
curl -k 'imaps://<IP>' --user <user>:<password>

# INBOX 메일 번호 전체
curl -k 'imaps://<IP>/INBOX?ALL' --user <user>:<password>

# 특정 메일 읽기
curl -k 'imaps://<IP>/INBOX;MAILINDEX=1' --user <user>:<password>

# 본문 검색
curl -k 'imaps://<IP>/INBOX?SEARCH BODY "pass"' --user <user>:<password>

# POP3 - 목록과 특정 메일
curl -k 'pop3s://<IP>' --user <user>:<password>
curl -k 'pop3s://<IP>/1' --user <user>:<password>
```

---

## 6. TLS 포트 직접 다루기 (993 / 995)

telnet 은 TLS 포트에 못 붙는다. openssl 로 암호화 세션을 열고, 그 안에서 위 5번 명령을 그대로 친다.

```bash
# IMAPS (993)
openssl s_client -connect <IP>:993 -crlf -quiet
a1 LOGIN <user> <password>
a2 SELECT INBOX
a3 FETCH 1 BODY[]

# POP3S (995)
openssl s_client -connect <IP>:995 -crlf -quiet
USER <user>
PASS <password>
RETR 1

# 평문 포트에서 STARTTLS 로 올려붙기 (110/143)
openssl s_client -connect <IP>:110 -starttls pop3
openssl s_client -connect <IP>:143 -starttls imap
```

---

## 7. 메일함에서 찾을 것

- 평문 비밀번호 - 초기 비번 안내 메일, 서비스 가입 확인 메일
- 다른 계정 힌트 - 동료 이메일 주소(사용자명 규칙), 내부 배포 리스트
- 내부망 정보 - 내부 IP, 호스트명, VPN/포털 링크, 첨부 설정 파일
- 재사용 가능성 - 메일 비번이 SSH/AD 등 다른 서비스에 재사용되는지 반드시 교차 시도

---

## 팁

- SMTP VRFY/RCPT 로 찾은 사용자 목록을 POP3/IMAP 계정 확인에 그대로 재활용한다
- 평문 인증 거부 서버는 USER 열거가 막히니, 계정 존재는 다른 서비스(SMTP, SSH 등)로 교차 확인한다
- telnet 은 110/143 전용이다. 993/995 는 openssl s_client 로 붙는다
- 메일 본문 검색은 IMAP SEARCH BODY 나 curl 이 빠르다. 메일이 많으면 하나씩 RETR 하지 말 것
- 확보한 메일 비번은 거의 항상 다른 서비스 재사용을 시도할 가치가 있다
