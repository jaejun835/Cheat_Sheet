# 08_DNS_열거

# DNS 열거

도메인명 ↔ IP 매핑 구조(zone)를 직접 조사해서 서브도메인, 내부 호스트명, AD 관련 서비스까지 찾아내는 기법.

---

## 서브도메인 열거가 필요한 이유

한 도메인 안에서도 서브도메인마다 다른 애플리케이션과 버전이 떠 있을 수 있다. 메인 사이트는 안전해도 dev 서브도메인에 취약점이 있는 경우가 흔하다. 그래서 IP 하나를 스캔한 걸로 정찰을 끝내면 안 되고, 서브도메인까지 별도로 찾아야 한다.

## 하위 도메인과 vhost의 차이

하위 도메인은 DNS에 등록된 이름이다. DNS는 이름을 IP로 변환해서 요청이 어디로 갈지 정한다. vhost는 그 요청이 도착한 뒤, 서버 설정 안에서 어떤 콘텐츠를 응답할지 정하는 것이다. 이 둘은 서로 다른 단계에서 일어나기 때문에 DNS에는 없고 vhost로만 존재하는 이름이 있을 수 있다. 둘 다 확인해야 하는 이유다.

## 동작 원리

1. 클라이언트가 `dev.example.com`을 요청한다.
2. DNS(또는 `/etc/hosts`)가 그 이름을 IP로 변환한다. 예: `10.10.10.15`.
3. 클라이언트가 그 IP로 TCP 연결을 맺고, HTTP 요청을 보낸다. 이때 요청 헤더에 `Host: dev.example.com`을 담는다.
4. 서버가 Host 헤더 값을 읽는다. 자기 설정 파일 안에 있는 여러 `ServerName` 값과 비교한다.

```
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/main
</VirtualHost>

<VirtualHost *:80>
    ServerName dev.example.com
    DocumentRoot /var/www/dev
</VirtualHost>
```

1. 일치하는 `ServerName`을 찾으면 그 블록의 `DocumentRoot`를 적용해서 응답한다. 일치하는 게 없으면 기본 vhost를 적용한다.

IP 주소는 유한하고 사이트마다 IP를 따로 할당하는 건 비효율적이라, IP 하나 포트 하나에 여러 사이트를 올려두고 Host 헤더로 구분하는 구조를 쓴다.

## 적용 범위

vhost는 HTTP/HTTPS에만 있는 개념이다. Host 헤더 자체가 HTTP 프로토콜 스펙에 정의된 헤더이기 때문이다. SMTP, FTP, SSH 같은 프로토콜에는 Host 헤더 개념이 없다. 그래서 ffuf/gobuster vhost 퍼징은 웹 서비스(80, 443, 8080 등)에서만 의미가 있고, 다른 포트에서는 DNS 열거만 해당된다.

## HTTPS에서 추가되는 단계

HTTPS는 TLS 암호화 때문에 연결 초기에는 서버가 Host 헤더를 볼 수 없다. 그런데 인증서는 도메인마다 다를 수 있어서, 어떤 인증서를 보여줄지를 Host 헤더보다 먼저 정해야 하는 문제가 생긴다. 이를 위해 TLS 계층에 SNI(Server Name Indication)라는 필드를 따로 두고, 암호화되기 전에 도메인 이름을 먼저 보낸다. TLS 연결이 끝난 뒤에야 HTTP 요청 안의 Host 헤더가 읽힌다. vhost를 최종 결정하는 건 여전히 Host 헤더이고, HTTPS는 그 전에 SNI로 인증서 선택 문제를 한 단계 먼저 처리하는 것뿐이다.

---

## 1단계: 기본 조회 (dig / host / nslookup)

```bash
# dig — 가장 상세한 출력, 실무에서 표준도구
dig example.com
# 기본값: A 레코드만 조회, 응답 전체(헤더+ANSWER+AUTHORITY+ADDITIONAL) 출력

dig +short example.com
# +short : IP값만 깔끔하게 출력 (스크립트에서 파싱할 때 유용)

dig A example.com      # A 레코드 (IPv4)
dig AAAA example.com   # AAAA 레코드 (IPv6)
dig MX example.com     # MX 레코드 (메일서버)
dig NS example.com     # NS 레코드 (네임서버)
dig TXT example.com    # TXT 레코드 (SPF, 인증토큰 등 텍스트 정보가 평문으로 들어있는 경우 많음)
dig SOA example.com    # SOA 레코드 (존(zone) 관리 정보, 관리자 메일 등)
dig ANY example.com
# ANY : 서버가 공개할 의향이 있는 모든 레코드를 한번에 요청 (최근 BIND는 기본 차단하는 추세라 안 나올 수도 있음)

dig -x <IP>
# -x : 역방향 조회 (PTR 레코드). IP로 도메인명 역추적

dig @<IP> example.com
# @<IP> : 이 DNS 서버에게 직접 질의 (타겟 자체의 DNS 서비스 조회는 이 방식)

# host — 간단한 조회
host example.com
host -t MX example.com
host <IP>
# 역방향 조회

# nslookup — 대화형도 가능
nslookup example.com
nslookup -type=MX example.com
nslookup example.com <IP>
# 특정 DNS 서버 지정

# 대화형 질의 (여러 IP 역추적할 때 편리)
nslookup
> SERVER <IP>
> <IP>
> exit
```

---

## 2단계: DNS 배너/버전 확인

```bash
dig version.bind CHAOS TXT @<IP>
# version.bind CHAOS TXT : BIND 계열 DNS 서버가 자기 버전을 노출하는 특수 쿼리 (보통 CHAOS 클래스를 쓰는 이유)
# 응답 예: "9.16.1-Ubuntu" → searchsploit로 바로 CVE 대조

fpdns -D <IP>
# fpdns : 버전 문자열이 가려져있거나 조작된 경우도 핑참 기반으로 소프트웨어 추출 시도

nmap -sU -p 53 --script=banner <IP>
# nmap banner 스크립트로도 간단히 시도 가능

# 배너 해석 주의사항
# 표준 버전 문자열(예: 9.16.1-Ubuntu)이 아니라 CUSTOM_STRING 같은 이상한 값이 나오면
# → 관리자가 의도적으로 바꿔놓은 것이라는 신호로 보고, 우연이 아니라 힌트일 가능성 먼저 의심
# → 문자열 그대로 구글링
# → zone transfer 재시도 (설정 자체가 이상할 가능성)
# → --version-intensity 9 로 nmap 재스캔 (커스텀 마이너 버전일 수도 있음)
```

---

## 3단계: Zone Transfer (AXFR) — 제일 먼저 시도해볼 것

존 전체를 복제하는 정상 메커니즘인데, 권한 없는 외부에게도 허용되면 존 전체(서브도메인, 내부 IP, 호스트명)를 한번에 유출하는 치명적 오설정이다.

```bash
dig axfr @<IP>
# 도메인 명 없이 시도 (NS 자체에 물어보는 방식)

dig axfr @<IP> <DOMAIN>
# 도메인명 지정해서 시도 (대부분 이 방식이 성공함)

dig NS <DOMAIN>
# 먼저 권한 서버(NS) 목록 확인 후

for ns in $(dig +short NS <DOMAIN>); do dig axfr @$ns <DOMAIN>; done
# 각 NS마다 순차적으로 zone transfer 시도

fierce --domain <DOMAIN> --dns-servers <IP>
# fierce : 권한 있는 모든 NS에 대해 zone transfer 자동 시도, 실패하면 자체 내장 사전으로 브루트포스 자동 전환

# 성공 예시 출력
# example.com.        3600  IN  NS    ns1.example.com.
# dev.example.com.     3600  IN  A     10.10.10.15
# gitlab.example.com.  3600  IN  A     10.10.10.20
# → 이거 하나로 ffuf/gobuster 수백번 돌릴 것을 한번에 얻는 셈
```

---

## 4단계: 자동화 도구로 종합 열거 (dnsrecon / dnsenum)

```bash
# dnsrecon — 가장 많이 쓰는 종합 도구, 여러 기능을 하나로 묶음
dnsrecon -d <DOMAIN>
# -d : 도메인 지정, 기본적으로 SOA/NS/A/AAAA/MX/TXT + zone transfer 자동 시도

dnsrecon -d <DOMAIN> -n <IP>
# -n : 조회할 네임서버 직접 지정 (내부망 타겟 DNS일 때 필수)

dnsrecon -d <DOMAIN> -a -n <IP>
# -a : AXFR(zone transfer) 시도 포함

dnsrecon -d <DOMAIN> -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t brt -n <IP>
# -t brt : 브루트포스 모드, 서브도메인 무작위대입
# -D : 사용할 워드리스트

dnsrecon -r <IP>/24 -n <IP>
# -r : 역방향 범위 전체 조회 (PTR 레코드 있는 호스트 찾기), 실제 대역으로 치환해서 사용

# dnsenum — dnsrecon과 유사, 출력이 조금 다름
dnsenum --dnsserver <IP> --enum -p 0 -s 0 -o subdomains.txt -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt <DOMAIN>
# -p 0 -s 0 : 구글 검색/whois 거치지 않고 순수 DNS 질의만
# -o : 결과 파일로 저장

# nmap 종합 DNS 스크립트
nmap -n -sU -p53 --script "(default and *dns*) or fcrdns or dns-srv-enum or dns-random-txid or dns-random-srcport" <IP>
```

---

## 5단계: 수동 서브도메인 브루트포스 (도구 없이 dig만으로)

```bash
for sub in $(cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt); do
  dig $sub.<DOMAIN> @<IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains-dig.txt
done
# grep -v ';\|SOA'  : 주석줄과 SOA 로그 제거
# sed 정규식     : 빈 줄 제거
# 결과가 있는 줄만 남아서 실제 존재하는 서브도메인만 추출됨
```

---

## 6단계: AD 환경에서 DNS SRV 레코드

Active Directory 도메인 컨트롤러는 자기 서비스를 DNS SRV 레코드로 광고한다. 크리덴셜 없이도 도메인 구조, 사이트 이름, 관리 범위를 파악할 수 있다.

```bash
dig -t SRV _ldap._tcp.dc._msdcs.<DOMAIN>
# _ldap._tcp.dc._msdcs : 도메인 컨트롤러(DC) 목록, 가장 먼저 확인하는 레코드

dig -t SRV _kerberos._tcp.<DOMAIN>
# Kerberos KDC 위치 → 포트, 호스트명 확인 (88번 포트 확인용)

dig -t SRV _kpasswd._tcp.<DOMAIN>
# Kerberos 비밀번호 변경 서비스 위치

dig -t SRV _gc._tcp.<DOMAIN>
# Global Catalog 서버 위치 (포레스트 전체 검색용, 보통 3268/3269 포트)

nslookup -type=srv _kerberos._tcp.<DOMAIN>
# nslookup으로도 동일하게 확인 가능

nmap --script dns-srv-enum --script-args "dns-srv-enum.domain='<DOMAIN>'" <IP>
# nmap으로 SRV 레코드 일괄 조회
```

AD 환경 판단 근거: SRV 레코드만 확인해도 도메인 컨트롤러 IP, Kerberos 포트 위치를 자격증명 없이 파악할 수 있다.

---

## 7단계: 찾은 서브도메인 등록 (이어서 ffuf/gobuster vhost 퍼징으로)

```bash
# 하나씩 등록
echo "<IP> example.htb" | sudo tee -a /etc/hosts

# 여러 개 한번에 등록 (heredoc 방식)
sudo tee -a /etc/hosts > /dev/null <<EOT
<IP> example.htb blog.example.htb dev.example.htb gitlab.example.htb
EOT
# tee -a          : append
# > /dev/null     : 화면 출력 숨김
# <<EOT ... EOT   : 여러 줄을 한번에 넘기는 heredoc
```

---

## 레코드타입별 의미 빠른 참고

| 레코드 | 의미 | 공격자 관점 활용법 |
| --- | --- | --- |
| A / AAAA | 도메인 → IP | 실제 호스트 발견, 새 스캔 대상 추가 |
| NS | 권한 DNS 서버 | zone transfer 시도할 대상 목록 |
| MX | 메일서버 | 내부 메일 인프라 노출, 피싱/SMTP 공격면 |
| TXT | 임의 텍스트 | SPF/DKIM, 때때로 인증토큰·설정값이 평문 노출 |
| SOA | 존 관리자 정보 | 관리자 메일 → 사용자명 추측 힌트 |
| SRV | 특정 서비스 위치 | AD/Kerberos/LDAP/GC 자동 발견 |
| PTR | IP → 도메인 | 역방향 조회로 내부망 호스트명 패턴 파악 |

---

## 도구 비교

| 도구 | 강점 | 약점 |
| --- | --- | --- |
| dig | 가장 상세, 스크립트 조합에 유리 | 수동 확인 번거로움 |
| dnsrecon | 종합 자동화(std+brt+axfr 한번에) | 옵션이 많아 처음엔 헷갈릴 수 있음 |
| fierce | zone transfer 자동 재시도 + 딕셔너리 공격 자동 전환 | 속도 조절 옵션 적음 |
| dnsenum | gnu 스타일 출력, 문서화 쉬움 | dnsrecon과 기능 겹침 |
| nmap NSE | 이미 돌린 스캔 결과에 바로 통합 | 전용 DNS 도구보다 기능이 제한적 |

---

## 팁

- 배너/버전 먼저 확인 (`version.bind CHAOS TXT`), 이상한 문자열이면 의도된 신호로 먼저 의심
- zone transfer 바로 시도 (`dig axfr`), 가장 가성비 높은 한 방
- 실패하면 자동화 도구로 종합 열거 (dnsrecon/fierce/dnsenum)
- AD 환경이 의심되면 SRV 레코드 별도 확인 (`_ldap`, `_kerberos`, `_gc`)
- 찾은 서브도메인은 `/etc/hosts` 등록 후 웹서버가 실제로 다르게 반응하는지 vhost 퍼징으로 재확인 (`03_gobuster_dirbuster` 참고)

---

## 새 타겟 시작 전 /etc/hosts 정리 (실수 재발 방지)

이전 타겟에서 등록해둔 옛날 IP 줄이 남아있으면, hosts 파일은 위에서부터 순서대로 매칭되기 때문에 **죽은 옛날 IP가 먼저 적용돼서 새 타겟이 타임아웃 나는 원인**이 된다.

```bash
# 현재 등록된 것 확인
cat /etc/hosts | grep <도메인키워드>

# 특정 옛날 IP가 포함된 줄만 삭제
sudo sed -i '/10.129.53.110/d' /etc/hosts

# 또는 특정 도메인 키워드가 들어간 줄 전부 삭제 (더 확실함)
sudo sed -i '/inlanefreight/d' /etc/hosts

# 정리 후 재확인
cat /etc/hosts | grep <도메인키워드>
```

**습관화할 것**: 새 타겟 잡을 때마다 `cat /etc/hosts`로 먼저 이전 세션 흔적이 있는지 확인하고, 있으면 지우고 새로 등록. 특히 같은 도메인 이름(HTB류 `inlanefreight.local` 같은 재사용 실습 도메인)을 여러 세션에 걸쳐 쓰는 경우 이 문제가 반복해서 생길 수 있음.