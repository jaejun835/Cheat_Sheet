# whois

# whois (수동 정찰)

도메인과 IP 의 등록 정보를 조회하는 수동 정찰 기법. 패킷을 타겟에 안 보내고 공개 등록 DB 를 질의하므로 흔적이 안 남는다. 소유 조직, 네임서버, IP 대역, 담당자 연락처를 얻어 공격 표면과 피벗 대상을 넓힌다. CPTS 의 footprinting 초반 단계.

---

## 1. 도메인 whois

```bash
whois <domain>
```

확인할 항목
- Registrar - 등록 대행사
- Registrant / Org - 등록자, 소유 조직 (프라이버시로 가려진 경우 많음)
- Creation / Expiry / Updated Date - 등록, 만료, 수정일
- Name Server - 네임서버 (DNS 열거로 이어짐)
- Status - 도메인 상태 (clientTransferProhibited 등)
- Admin / Tech Contact - 관리/기술 담당자 이메일, 전화 (피싱/OSINT 단서)

---

## 2. IP whois

IP 를 질의하면 해당 RIR(지역 인터넷 등록기관)의 정보가 나온다. 소유 조직과 IP 대역을 알 수 있다.

```bash
whois <IP>
```

확인할 항목
- NetRange / CIDR - 이 IP 가 속한 대역 (추가 타겟 범위)
- Organization - 대역 소유 조직
- OriginAS - 소속 ASN
- abuse 연락처 - 조직 이메일 도메인 단서

---

## 3. ASN 조회

조직이 보유한 전체 IP 대역을 찾을 때 ASN 으로 역추적한다.

```bash
whois -h whois.radb.net -- '-i origin AS<number>'   # ASN 이 광고하는 대역 목록
whois AS<number>                                     # ASN 기본 정보
```

---

## 4. whois 서버 지정

```bash
# 특정 whois 서버로 질의
whois -h <whois_server> <query>

# 예: ARIN 직접 질의
whois -h whois.arin.net <IP>
```

---

## 5. RDAP (whois 후속 프로토콜)

RDAP 은 whois 를 대체하는 구조화(JSON) 프로토콜이다. 파싱이 쉬워 자동화에 유리하다.

```bash
# 도메인
curl -s https://rdap.org/domain/<domain> | jq

# IP
curl -s https://rdap.org/ip/<IP> | jq
```

---

## 팁

- 네임서버와 담당자 이메일 도메인은 DNS 열거(08)와 서브도메인 탐색으로 바로 이어진다
- IP whois 의 NetRange 로 같은 조직의 추가 호스트 대역을 넓힐 수 있다
- 등록자 정보가 WHOIS privacy 로 가려져 있어도, 과거 기록(whois history, 유료)이나 인증서(crt.sh)로 우회 가능
- 흔적이 안 남는 수동 정찰이므로 능동 스캔 전에 먼저 돌린다