# 02_OSINT_검색엔진

# OSINT / 검색엔진 디스커버리

패킷을 타겟에 안 보내고 공개 정보만 수집하는 수동 정찰. 흔적이 안 남아 능동 스캔 전에 먼저 돌린다. 검색엔진 연산자로 노출된 민감 정보를 찾고, 과거 스냅샷으로 사라진 것을 복원하고, 전용 도구로 이메일/서브도메인/노출 자산을 모은다.

---

## 1. Google Dorking (검색 연산자)

검색엔진의 특수 연산자로 공개된 민감 정보를 저격한다. 연산자 조합이 핵심이다.

```
site:         특정 도메인으로 한정        site:example.com
inurl:        URL 에 특정 단어            inurl:login
intitle:      제목에 특정 단어            intitle:"index of"
intext:       본문에 특정 단어            intext:"password"
filetype:     특정 파일 형식             filetype:pdf
ext:          확장자 (filetype 과 유사)   ext:conf
cache:        구글 캐시 버전              cache:example.com
"..."         정확한 문구                "internal use only"
-             제외                       site:example.com -inurl:www
*             와일드카드                  site:example.com user* manual
AND OR NOT    조합                       site:example.com (inurl:admin OR inurl:login)
```

### 실전 dork

```
# 로그인/관리 페이지
site:<target> (inurl:login OR inurl:admin OR inurl:portal)

# 노출된 문서/파일
site:<target> (filetype:pdf OR filetype:xls OR filetype:docx)
site:<target> filetype:txt (intext:password OR intext:user)

# 설정/백업 파일
site:<target> (ext:conf OR ext:cnf OR ext:ini OR ext:env)
site:<target> (inurl:backup OR filetype:sql OR ext:bak)

# 디렉터리 리스팅
site:<target> intitle:"index of"

# 에러/정보 노출
site:<target> intext:"sql syntax near" OR intext:"Warning: mysql"
```

GHDB(Google Hacking Database, exploit-db.com/google-hacking-database)에 다량의 dork 가 정리돼 있다.

---

## 2. Wayback Machine (과거 스냅샷)

Internet Archive(web.archive.org)가 보관한 과거 웹페이지. 지금은 사라졌지만 과거엔 노출됐던 것을 찾는 게 공격 관점의 핵심이다.

```
# 날짜 지정 직접 접속 (달력 탐색 생략)
http://web.archive.org/web/<YYYYMMDD>/https://<target>/
http://web.archive.org/web/19991012/http://paypal.com/     # 예: 1999-10-12
```

침투 관점에서 찾을 것
- 과거에만 있던 관리자 경로, 삭제된 API 엔드포인트
- URL 에 박혀 있던 파라미터 (현재는 사라진 것)
- 옛 직원 이메일, 연락처, 내부 문서 링크
- 과거 기술스택/버전 (현재 서버의 레거시 흔적)

```bash
# 특정 도메인의 저장된 URL 전부 긁기 (파라미터/엔드포인트 발굴)
curl -s "http://web.archive.org/cdx/search/cdx?url=<target>/*&output=text&fl=original&collapse=urlkey" | sort -u

# waybackurls 도구 (같은 일을 더 편하게)
echo <target> | waybackurls | sort -u
```

주의 - Wayback 문제/값은 화면에서 복사-붙여넣기할 것. 과거 페이지엔 눈에 안 띄는 오타/특수문자가 있어, 손으로 타이핑하면 원문과 안 맞는 경우가 있다.

---

## 3. 이메일 / 서브도메인 수집 (theHarvester)

```bash
# 여러 공개 소스에서 이메일/서브도메인/호스트 수집
theHarvester -d <target> -b all
theHarvester -d <target> -b bing,duckduckgo,crtsh -l 500

# 결과 활용: 이메일 → 사용자명 규칙 → 로그인 스프레이 대상
```

---

## 4. 서브도메인 - 인증서 투명성 로그 (crt.sh)

발급된 SSL 인증서 기록에서 서브도메인을 뽑는다. WHOIS privacy 로 가려져도 인증서는 남는다.

```bash
# crt.sh JSON 파싱
curl -s "https://crt.sh/?q=%25.<target>&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u
```

---

## 5. 노출 자산 검색 (Shodan / Censys)

인터넷에 노출된 호스트/서비스/배너를 검색하는 엔진. 직접 스캔 없이 열린 포트, 서비스 버전, 취약 기기를 찾는다.

```
# Shodan 검색 예 (웹 또는 CLI)
hostname:<target>
org:"<조직명>"
ssl:"<target>"
port:3389 country:KR
product:"Apache httpd"
```

- Shodan - shodan.io, CLI `shodan search`, `shodan host <IP>`
- Censys - search.censys.io, 인증서/호스트 중심

---

## 팁

- OSINT 는 흔적이 안 남는다. 능동 스캔(nmap 등) 전에 먼저 돌려 공격 표면을 넓힌다
- Google Dorking 으로 노출된 파일/설정/로그인 페이지를 먼저 훑는다
- Wayback 은 "지금 없는" 엔드포인트/파라미터의 보고. waybackurls 로 자동 수집
- crt.sh + theHarvester 로 서브도메인/이메일을 모아 04(vhost)와 로그인 공격으로 연결
- 수집한 이메일 형식(firstname.lastname@)은 사용자명 규칙 추정에 직결