# 02_WAF탐지_우회

# WAF 탐지 / 우회 (wafw00f)

공격을 시작하기 전에 WAF(Web Application Firewall) 유무를 먼저 확인한다. WAF 가 있으면 SQLi/XSS 페이로드가 차단되고 디렉터리 브루트포스로 IP 가 밴당할 수 있다. 모르고 들이박으면 시간도 날리고 접근까지 막히므로, 공격 전 판단 포인트다.

---

## 1. 기본 사용

```bash
# 기본 탐지
wafw00f http://<IP>

# 모든 WAF 탐지 (하나 찾아도 멈추지 않고 전부 확인)
wafw00f http://<IP> -a

# 탐지 가능한 WAF 목록
wafw00f -l

# 여러 대상 + 저장
wafw00f -i urls.txt -o wafw00f.txt
```

---

## 2. 결과 해석

- `is behind <WAF명>` - 해당 WAF 가 앞단에 있다. 이후 공격 전략을 바꾼다
- `No WAF detected` - 탐지 안 됨. 다만 없다는 보장은 아니다 (미탐 가능)
- wafw00f 하나만 믿지 말 것. 응답 헤더/쿠키(예: `__cfduid`, `cf-ray` = Cloudflare)나 nmap 스크립트로 교차 확인

---

## 3. WAF 교차 확인

```bash
# nmap WAF 스크립트
nmap -p 80,443 --script=http-waf-detect <IP>
nmap -p 80,443 --script=http-waf-fingerprint <IP>

# 응답 헤더에서 WAF 흔적 확인
curl -sI http://<IP> | grep -iE 'server|x-powered|cf-ray|x-sucuri|x-waf'
```

---

## 4. WAF 감지됐을 때 대응 (판단)

구체적 페이로드 우회는 각 공격 파일(SQLi/XSS)에 있다. 여기선 전술 판단만.

- 속도 낮추기 - 브루트포스/퍼징 시 스레드 줄이고 딜레이 추가 (IP 밴 회피)
  - gobuster `-t 5 --delay 200ms`, ffuf `-rate`, `-p` 딜레이
- 출처 위장 헤더 - 내부 요청인 척
  - `X-Forwarded-For: 127.0.0.1`, `X-Originating-IP: 127.0.0.1`, `X-Remote-Addr: 127.0.0.1`
- 페이로드 인코딩/변형 - WAF 룰 회피 (상세는 SQLi/XSS 파일 참고)
  - URL 인코딩, 대소문자 섞기, 주석 삽입, 더블 인코딩
- User-Agent 변경 - 스캐너 기본 UA 는 바로 차단되는 경우 많음
- IP 로테이션 - 밴 지속되면 출구 IP 변경 (프록시/VPN)
- 직접 오리진 접근 - CDN/WAF 뒤의 실제 서버 IP 를 찾으면 WAF 를 우회 (과거 DNS 기록, SSL 인증서, 서브도메인에서 유출된 IP)

---

## 팁

- 공격 전 WAF 확인은 필수. 디렉터리 브루트포스/SQLi 로 밴당하기 전에 먼저 본다
- wafw00f 미탐 가능성 때문에 nmap 스크립트와 응답 헤더로 교차 확인
- WAF 가 있으면 "느리게 + 변형해서", 없으면 "빠르게" 로 전략이 갈린다
- CDN 뒤 실제 오리진 IP 를 찾으면 WAF 를 통째로 우회할 수 있다