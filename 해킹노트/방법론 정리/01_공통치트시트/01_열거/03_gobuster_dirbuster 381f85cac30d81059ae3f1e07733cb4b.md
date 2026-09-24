# 03_gobuster_dirbuster

# gobuster / feroxbuster / ffuf

웹 디렉터리·서브도메인·vhost 등을 wordlist 기반으로 brute-force 탐색.

---

## gobuster — dir 모드

```bash
############################################
# 1. 기본 디렉터리 탐색
############################################

# 가장 기본형
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt

# OSCP 기본 추천형: 안정성 우선
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -o gobuster-common.txt

# 느린 VPN / timeout 많을 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 5 \
  --timeout 30s \
  -o gobuster-slow.txt

# 빠른 환경일 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 30 \
  --timeout 10s \
  -o gobuster-fast.txt

############################################
# 2. 상태 코드 필터링
############################################

# 404, 400 제외
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  -b 404,400 \
  -o gobuster-filter-status.txt

# 404, 400, 403 제외
# 주의: 403 중에 중요한 경로가 있을 수도 있어서 초반엔 조심
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  -b 404,400,403 \
  -o gobuster-no403.txt

# 특정 status만 보고 싶을 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  -s 200,204,301,302,307,401,403 \
  -o gobuster-status-allow.txt

############################################
# 3. 응답 크기 필터링
############################################

# 특정 크기 제외
# 예: nginx dotfile 403이 전부 Size 162로 뜰 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  --exclude-length 162 \
  -o gobuster-exclude-size.txt

# 여러 크기 제외
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --exclude-length 162,9265 \
  -o gobuster-exclude-sizes.txt

############################################
# 4. 디렉터리 워드리스트별 탐색
############################################

# 빠른 1차 정찰
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-quick.txt

# small directories
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-small-dirs.txt

# medium directories
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-medium-dirs.txt

############################################
# 5. 파일 탐색
############################################

# 파일명이 이미 포함된 files 리스트 사용
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-files.txt

# common 파일 탐색
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-common-files.txt

############################################
# 6. 확장자 탐색
############################################

# PHP, TXT 확장자 탐색
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-ext-basic.txt

# PHP 웹앱에서 자주 쓰는 확장자
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,txt,bak,old,zip \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-ext-php.txt

# ASP / IIS 의심될 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x asp,aspx,txt,bak,old \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-ext-asp.txt

# JSP / Tomcat 의심될 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x jsp,txt,bak,old \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-ext-jsp.txt

############################################
# 7. 발견한 경로 집중 탐색
############################################

# /admin 발견 후 추가 탐색
gobuster dir -u http://<IP>/admin/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-admin.txt

# /api 발견 후 추가 탐색
gobuster dir -u http://<IP>/<api>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x <api>
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-api.txt

# /uploads 발견 후 추가 탐색
gobuster dir -u http://<IP>/uploads/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-uploads.txt

# /js 발견 후 JS 파일 탐색
gobuster dir -u http://<IP>/js/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x js \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-js.txt

############################################
# 8. HTTPS / 인증서 무시
############################################

# HTTPS 기본
gobuster dir -u https://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-https.txt

# HTTPS 인증서 검증 무시
gobuster dir -u https://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -k \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-https-k.txt

############################################
# 9. Host 헤더 / 도메인 기반 웹
############################################

# /etc/hosts 등록 후 도메인으로 탐색
echo "<IP> example.htb" | sudo tee -a /etc/hosts

gobuster dir -u http://example.htb/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-domain.txt

# Host 헤더 직접 지정
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -H "Host: example.htb" \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-host-header.txt

############################################
# 10. 인증 / 쿠키 / 헤더
############################################

# Cookie 사용
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -c "PHPSESSID=abc123; session=xyz" \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-cookie.txt

# Authorization Bearer 토큰
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -H "Authorization: Bearer <TOKEN>" \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-bearer.txt

# Basic Auth
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -U <USERNAME> \
  -P <PASSWORD> \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-basic-auth.txt

# 여러 헤더 추가
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -H "Cookie: session=abc" \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "X-Originating-IP: 127.0.0.1" \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-headers.txt

############################################
# 11. User-Agent / Redirect
############################################

# User-Agent 변경
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -a "Mozilla/5.0" \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-useragent.txt

# Redirect 따라가기
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -r \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-redirect.txt

# 전체 URL 출력
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -e \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-expanded.txt

############################################
# 12. 프록시 / Burp 연결
############################################

# Burp 프록시 사용
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --proxy http://127.0.0.1:8080 \
  -t 5 \
  --timeout 30s \
  -b 404,400 \
  -o gobuster-burp.txt

# HTTPS + Burp + 인증서 무시
gobuster dir -u https://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  --proxy http://127.0.0.1:8080 \
  -k \
  -t 5 \
  --timeout 30s \
  -b 404,400 \
  -o gobuster-https-burp.txt

############################################
# 개념 먼저: DNS 서브도메인 vs vhost 차이
############################################
# DNS 열거(dig/dnsrecon) = "이 이름이 DNS에 공식 등록되어 있는가"
# vhost 퍼징(gobuster vhost/ffuf) = "DNS에 있든 없든 웹서버가 이 Host 헤더값을 인식해서 다른 콘텐츠를 보여주는가"
# 대부분 겹치지만, 관리자가 DNS에는 안 올리고 자기들만 /etc/hosts로 쓰는 숨겨진 개발/관리자 vhost가
# 실제 취약점이 제일 많이 나오는 지점이라, DNS 열거(08_DNS_열거 참고)를 먼저 하고 이어서 아래 vhost 퍼징을 반드시 병행할 것

############################################
# 13. VHOST / 서브도메인 탐색
############################################

# vhost 탐색 기본
gobuster vhost -u http://example.htb/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 10 \
  --timeout 20s \
  -o gobuster-vhost.txt

# append-domain 사용
gobuster vhost -u http://example.htb/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -t 10 \
  --timeout 20s \
  -o gobuster-vhost-append.txt

# 응답 크기로 필터링
gobuster vhost -u http://example.htb/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  --exclude-length <DEFAULT_SIZE> \
  -t 10 \
  --timeout 20s \
  -o gobuster-vhost-filter.txt

############################################
# 14. DNS 서브도메인 탐색
############################################

# DNS 모드
gobuster dns -d example.htb \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 20 \
  -o gobuster-dns.txt

# 와일드카드 DNS 강제 진행
gobuster dns -d example.htb \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 20 \
  --wildcard \
  -o gobuster-dns-wildcard.txt

############################################
# 15. Fuzz 모드
############################################

# FUZZ 위치 직접 지정
gobuster fuzz -u http://<IP>/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-fuzz-path.txt

# 파라미터 이름 fuzzing
gobuster fuzz -u "http://<IP>/index.php?FUZZ=test" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -t 10 \
  --timeout 20s \
  -o gobuster-fuzz-param.txt

# 파라미터 값 fuzzing
gobuster fuzz -u "http://<IP>/index.php?id=FUZZ" \
  -w /usr/share/seclists/Fuzzing/special-chars.txt \
  -t 5 \
  --timeout 30s \
  -o gobuster-fuzz-value.txt

############################################
# 16. 백업 / 민감 파일 탐색
############################################

# 백업 확장자
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x bak,old,backup,zip,tar,gz,7z,sql,txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-backups.txt

# PHP 소스 백업 의심
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php.bak,php.old,php.save,php.txt,phps \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-php-backups.txt

# 민감 파일 위주
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-quickhits.txt

############################################
# 17. 느린 VPN / 불안정 서버용 추천 세트
############################################

# 가장 안전한 gobuster
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 5 \
  --timeout 30s \
  --exclude-length 162 \
  -o gobuster-safe.txt

# 더 느릴 때
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 2 \
  --timeout 45s \
  --exclude-length 162 \
  -o gobuster-very-slow.txt

# 딜레이 추가
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 5 \
  --delay 200ms \
  --timeout 30s \
  --exclude-length 162 \
  -o gobuster-delay.txt

############################################
# 18. OSCP 추천 루틴
############################################

# Step 1: 빠른 루트 정찰
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-01-common.txt

# Step 2: small directories
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-02-small-dirs.txt

# Step 3: 파일 리스트
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-03-files.txt

# Step 4: 확장자 탐색
gobuster dir -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x php,txt,bak,old \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-04-ext.txt

# Step 5: 발견 경로만 집중 탐색
gobuster dir -u http://<IP>/<FOUND_PATH>/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10 \
  --timeout 20s \
  -b 404,400 \
  -o gobuster-05-found-path.txt

############################################
# 19. 지금 Intentions 같은 상황용
############################################

# 403 Size 162 노이즈 제거
gobuster dir -u http://10.129.229.27/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 5 \
  --timeout 30s \
  --exclude-length 162 \
  -o gobuster-intentions-root.txt

# /js 집중
gobuster dir -u http://10.129.229.27/js/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -x js \
  -t 5 \
  --timeout 30s \
  --exclude-length 162 \
  -o gobuster-intentions-js.txt

# /admin 확인용
gobuster dir -u http://10.129.229.27/admin/ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 5 \
  --timeout 30s \
  --exclude-length 162 \
  -o gobuster-intentions-admin.txt

############################################
# 20. 유용한 사전 확인 명령
############################################

# 웹 응답 헤더 확인
curl -i http://<IP>/

# 리다이렉트 따라가기
curl -iL http://<IP>/admin

# 응답 시간 확인
for i in {1..10}; do curl -o /dev/null -s -w "%{time_connect} %{time_starttransfer} %{time_total}\n" http://<IP>/; done

# 현재 403/404 크기 확인
curl -i http://<IP>/doesnotexist
curl -i http://<IP>/.git/config

# HTML에서 JS/CSS 경로 추출
curl -s http://<IP>/ | grep -Eo 'src="[^"]+|href="[^"]+' | sort -u
```

## gobuster — dns 모드 (서브도메인)

```bash
gobuster dns -d example.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -t 50 -i

# 와일드카드 강제
gobuster dns -d example.com -w wordlist.txt --wildcard -i
```

## gobuster — vhost 모드

```bash
gobuster vhost -u http://<IP> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain -t 50 -o vhosts.txt
```

---

## feroxbuster (재귀 자동 — OSCP 강력 추천)

gobuster는 비재귀 → 발견된 디렉터리 하위를 자동으로 안 파고듦. feroxbuster는 자동 재귀.

```bash
# 빠른 1차 정찰: 재귀/확장자 없이 표면 경로만
feroxbuster -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt \
  -t 30 -n \
  -C 404,400 \
  -o ferox-quick.txt
  

# 기본 디렉터리/라우트 탐색: 초반 추천
feroxbuster -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 30 -n \
  -C 404,400 \
  -o ferox-dirs.txt
  
# 파일 탐색: 확장자가 이미 포함된 워드 리스트 사용
feroxbuster -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -t 30 -n \
  -C 404,400 \
  -o ferox-files.txt
  
# 확장자 붙여서 탐색: words 리스트와 조합
feroxbuster -u http://<IP>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
  -x php,txt \
  -t 30 -n \
  -C 404,400 \
  -o ferox-ext.txt
 
# 더 깊은 재귀: 시간이 오래 걸리므로 필요한 경로에만
feroxbuster -u http://<IP>/uploads/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 25 -d 2 \
  -C 404,400 \
  -o ferox-recursive.txt 
  
# 발견 경로 집중 탐색
feroxbuster -u http://<IP>/<FOUND_PATH>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
  -t 30 -d 1 \
  -C 404,400 \
  -o ferox-<FOUND_PATH>.txt
  
# 필터: 404 제거 + 특정 응답 크기 제거
feroxbuster -u http://<IP>/ \
  -w wordlist.txt \
  --filter-status 404 \
  --filter-size 9265
  
# 필터: 여러 status 제거
feroxbuster -u http://<IP>/ \
  -w wordlist.txt \
  -C 404,400,403
  
# 인증 헤더 / 쿠키
feroxbuster -u http://<IP>/ \
  -w wordlist.txt \
  -H "Cookie: session=abc" \
  -H "Authorization: Bearer xxx"
  
# User-Agent 변경
feroxbuster -u http://<IP>/ \
  -w wordlist.txt \
  -A "Mozilla/5.0"
  
# 결과 저장
feroxbuster -u http://<IP>/ \
  -w wordlist.txt \
  -o ferox.txt
  
# JSON 결과 저장
feroxbuster -u http://<IP>/ \
  -w wordlist.txt \
  --json \
  -o ferox.json
 
____________________________________________________________________________/

# 집중 재귀 
feroxbuster -u http://<IP> \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,txt,bak -t 50 -d 3

# 필터
feroxbuster -u http://<IP> -w wordlist.txt \
  --filter-status 404 --filter-size 9265

# 인증 헤더
feroxbuster -u http://<IP> -w wordlist.txt \
  -H "Cookie: session=abc" -H "Authorization: Bearer xxx"

# 결과 저장
feroxbuster -u http://<IP> -w wordlist.txt -o ferox.txt

# 재개 (중간에 멈춘 경우)
feroxbuster --resume-from ferox.txt
```

---

## ffuf (가장 유연한 fuzzer)

```bash
# 디렉터리 fuzz
ffuf -u http://<IP>/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,204,301,302,307,401,403

# 응답 크기 필터
ffuf -u http://<IP>/FUZZ -w wordlist.txt -fs 1234

# 확장자 fuzz
ffuf -u http://<IP>/indexFUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt

# vhost fuzz
ffuf -u http://<IP> \
  -H "Host: FUZZ.example.com" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 1234

# 파라미터 fuzz
ffuf -u "http://<IP>/page.php?FUZZ=test" -w params.txt -fc 404

# POST 로그인 브루트포스
ffuf -u http://<IP>/login -X POST \
  -d "username=admin&password=FUZZ" \
  -w /usr/share/wordlists/rockyou.txt \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fc 401 -fs 1234

# JSON POST fuzz
ffuf -u http://<IP>/api/login -X POST \
  -d '{"username":"admin","password":"FUZZ"}' \
  -H "Content-Type: application/json" \
  -w wordlist.txt -fc 401

# 결과 저장
ffuf -u http://<IP>/FUZZ -w wordlist.txt -o ffuf.json -of json
```

---

## 도구 비교

| 도구 | 강점 | 약점 |
| --- | --- | --- |
| gobuster | 빠름, 안정적, 다용도 | 비재귀 |
| dirbuster | GUI | 느리고 불안정 |
| feroxbuster | 자동 재귀, 빠름 | 옵션 학습 곡선 |
| ffuf | 헤더/POST/JSON 가장 유연 | 필터 수동 설정 |

---

## 워드리스트 추천

```bash
# 빠른 초기
/usr/share/seclists/Discovery/Web-Content/common.txt

# 표준
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# 정밀
/usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt

# API 엔드포인트
/usr/share/seclists/Discovery/Web-Content/api/objects.txt
```

---

## 경로 탐색 결과에서 중요한 것 정리

```bash
# 상태 코드

200    → 정상 접근 가능. 내용 바로 확인
301    → 영구 리다이렉트. 어디로 튕기는지 확인
302    → 임시 리다이렉트. 로그인 없으면 튕기는 경우 많음 → 인증 후 재시도
403    → 파일/디렉터리 존재하지만 접근 차단. 단순히 스킵하지 말고 우회 시도
           우회 방법: URL 끝에 / 추가, 대소문자 변환, X-Forwarded-For: 127.0.0.1 헤더 추가
401    → 인증 필요. 크리덴셜 확보 후 반드시 재시도
500    → 서버 에러. 입력값 처리 중 터진 것 → 취약점 힌트. 파라미터 조작 시도
404    → 없음. 스킵
____________________________________________________________________________/

# 최우선

/admin
→ 관리자 패널. 302로 튕기면 로그인 필요 또는 인증 우회 시도
→ 크리덴셜 확보되면 바로 재접근. admin 전용 기능, 파일 업로드, 유저 관리 기능 존재 여부 확인
→ 일반 유저 토큰으로 admin 엔드포인트 접근 가능한지 반드시 시도 (인가 취약점)

/.env
→ Laravel/Node/Django 등에서 DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD,
   APP_KEY, JWT_SECRET, API_KEY 등 민감 정보 직접 노출
→ 발견 즉시 전체 내용 확인. 크리덴셜 재사용 여부 SSH/DB 전부 시도

/.git
→ 소스코드 전체 복원 가능
→ 외부 접근 가능하면 git-dumper http://<IP>/.git/ ./dump 로 전체 덤프
→ 내부 쉘 얻은 후엔 git log --oneline --all 로 전체 커밋 히스토리 확인
→ git show <HASH>:파일경로 로 삭제된 크리덴셜, 하드코딩된 비밀번호, 테스트 계정 복원 가능
→ 개발자가 테스트용으로 커밋에 박아놓은 비밀번호가 실제 운영 계정에 그대로 쓰이는 경우 많음
→ 덤프한 저장소 전체에서 도메인 패턴으로 검색하면 이메일:비밀번호 쌍을 더 잘 찾을 수 있음: grep -ir "@<대상도메인>" 2>/dev/null (예: grep -ir "@dog.htb")

/upload, /uploads
→ 파일 업로드 기능 존재 → 웹셸 업로드 → RCE 시도
→ 확장자 필터 우회: .php5 .phtml .phar .php.jpg .php%00.jpg 등 시도
→ Content-Type 변조: image/jpeg 로 설정하고 PHP 코드 삽입
→ 업로드 후 /storage, /uploads, /files 등에서 직접 접근 가능한지 확인
→ 접근 가능하면 cmd 파라미터로 명령 실행 확인

/login
→ SQLi 시도. 에러 메시지 차이로 blind 여부 판단
→ ' 입력 후 500 에러 또는 DB 에러 메시지 뜨면 SQLi 가능성 높음
→ 크리덴셜 브루트포스 가능 여부 확인. rate limit, 계정 잠금 정책 있는지 확인
→ 기본 크리덴셜 시도: admin/admin, admin/password, root/root 등

/api
→ JS 파일과 함께 분석해서 엔드포인트 구조 파악
→ v1/v2 버전 차이 확인. 구버전 API에 인증 없이 접근 가능한 경우 있음
→ admin 전용 엔드포인트 존재 여부 확인
→ 일반 토큰으로 admin 엔드포인트 접근 가능한지 반드시 시도 (인가 취약점)
→ grep -RhoE "/api/[A-Za-z0-9_./?&=%:{}-]+" *.js | sort -u
____________________________________________________________________________/

# 중요

/js
→ app.js, admin.js 등 다운받아서 API 엔드포인트 전부 추출
→ 인증 토큰 처리 방식, admin 전용 라우트, 숨겨진 파라미터 파악
→ 하드코딩된 API 키, 시크릿, 엔드포인트 주석 확인

/storage
→ Laravel에서 업로드된 파일 저장 위치
→ 웹셸 업로드 성공하면 여기서 직접 실행 가능한지 확인

/backup
→ 소스코드, DB 덤프 파일 노출 가능성
→ .zip .tar .gz .sql .bak .old 확장자로 직접 접근 시도
→ 사이트명, 도메인명 기반으로 파일명 추측해서 시도

/config
→ 설정 파일 직접 노출 가능성. DB 접속 정보, API 키, 암호화 키 등 포함 가능
→ config.php, config.yml, config.json, settings.py 등 직접 접근 시도

/phpinfo.php
→ PHP 버전, 설치 경로, 활성화된 모듈, 환경변수, 설정값 전체 노출
→ disable_functions 확인 → 웹셸에서 사용 가능한 함수 파악
→ 버전 확인 후 해당 버전 CVE 탐색

/robots.txt, /sitemap.xml
→ 크롤러에게 숨기려고 명시한 경로가 오히려 힌트
→ Disallow 항목 전부 직접 접근 시도
____________________________________________________________________________/

# 참고

/vendor
→ PHP Composer 패키지 목록. composer.json, composer.lock 확인
→ 패키지 버전 확인 후 알려진 CVE 탐색

/node_modules
→ JS 패키지 버전 확인. package.json, package-lock.json 확인
→ 취약한 버전이면 CVE 탐색

/css, /fonts, /images
→ 보통 중요하지 않음. 프레임워크, 라이브러리 버전 힌트 정도
```