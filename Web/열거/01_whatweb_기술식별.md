# 01_whatweb_기술식별

# whatweb (웹 기술 식별)

웹이 나오면 가장 먼저 "뭘로 돌아가는가"를 파악한다. whatweb 은 한 번 요청해 응답의 헤더/쿠키/HTML/메타태그를 분석하고, 웹서버, OS, CMS, 프레임워크, 언어를 자동 식별한다. 기술이 특정되면 그에 맞는 공격(CMS별 익스플로잇 등)으로 바로 넘어간다.

---

## 1. 기본 사용

```bash
# 기본
whatweb http://<IP>

# 공격성 레벨 (-a 1~4, 숫자 클수록 요청 많고 정보 많음)
whatweb -a 3 http://<IP>      # 3 이 정보/부담 균형점. 자주 씀
#   1: stealthy (요청 1회, 가벼움)
#   3: aggressive (추가 요청으로 더 깊이 식별)
#   4: heavy (가장 많이 찌름)

# 상세 출력
whatweb -v http://<IP>

# 여러 대상 + 저장
whatweb -i urls.txt --log-brief=whatweb.txt
whatweb http://<IP> --log-json=whatweb.json
```

---

## 2. 결과 판독 (핵심)

출력은 `항목[값]` 나열이다. 공격에 쓸 핵심 항목만 골라 본다.

```
http://app.inlanefreight.local [200 OK] Apache[2.4.41], Bootstrap, Cookies[...],
HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], JQuery,
MetaGenerator[Joomla! - Open Source Content Management], Title[Home]
```

- `HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)]` - OS 와 웹서버. 괄호 안 배포판(Ubuntu/Debian)이 OS, Apache 버전으로 CVE 매핑
- `MetaGenerator[Joomla! ...]` - CMS. generator 메타태그에서 읽음. WordPress/Joomla/Drupal 여기서 특정
- `Apache[2.4.41]` / `nginx` / `Microsoft-IIS` - 웹서버와 버전 → searchsploit 으로 CVE
- `X-Powered-By[PHP/7.x]` / `PoweredBy` - 백엔드 언어/버전
- `Bootstrap`, `JQuery` - 프론트 프레임워크 (버전 노출 시 취약 라이브러리 확인)
- `Cookies[...]`, `HttpOnly` - 세션 쿠키 이름으로 기술 추정 (예: PHPSESSID=PHP, JSESSIONID=Java)
- `WordPress[버전]`, 플러그인/테마 - CMS 상세 (나오면 wpscan 으로)
- `Title`, `Email`, `IP` - 부가 정보

판독 요령
- OS 는 HTTPServer 괄호 안, CMS 는 MetaGenerator 를 본다 (둘을 혼동 말 것 - Ubuntu 는 OS, Joomla 는 CMS)
- 버전이 보이는 모든 항목은 searchsploit/CVE 검색 대상

---

## 3. vhost / 도메인 대상

도메인으로 식별하려면 먼저 /etc/hosts 에 등록해야 한다. 안 그러면 "Name or service not known" 이 난다.

```bash
echo "<IP> app.example.local dev.example.local" | sudo tee -a /etc/hosts

whatweb -a 3 http://app.example.local
whatweb -a 3 http://app.example.local:<port>   # 비표준 포트면 포함
```

vhost 마다 OS/CMS 가 다를 수 있으니 각각 돌려 확인한다.

---

## 4. 다음 단계 연결

식별 결과에 따라 분기한다.

- CMS = WordPress → wpscan, WordPress 공격 파일
- CMS = Joomla → joomscan, Joomla 공격 파일
- CMS = Drupal → droopescan, Drupal 공격 파일
- 웹서버/언어 버전 노출 → searchsploit <제품> <버전>
- WAF 흔적(쿠키/헤더) → 02_WAF탐지_우회

---

## 보조 - 수동 식별

whatweb 이 못 잡거나 교차 확인할 때.

```bash
# generator 메타태그 직접 확인 (CMS)
curl -s http://<IP>/ | grep -i generator

# 응답 헤더 (서버/언어)
curl -sI http://<IP>/ | grep -iE 'server|x-powered-by'

# CMS 특유 경로로 확인
curl -s http://<IP>/administrator/   # Joomla 관리자
curl -s http://<IP>/wp-login.php      # WordPress
curl -s http://<IP>/user/login        # Drupal
```