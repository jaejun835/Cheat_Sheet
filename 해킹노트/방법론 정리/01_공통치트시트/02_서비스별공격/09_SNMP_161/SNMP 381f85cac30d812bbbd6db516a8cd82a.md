# SNMP

# SNMP (161 UDP)

네트워크 장비 관리 프로토콜. 기본 community string으로 시스템/네트워크 정보 대량 수집 가능.

---

## 1. 열거

```bash
# nmap (UDP)
nmap -sU -p 161 --script=snmp-info,snmp-sysdescr,snmp-processes,snmp-netstat,snmp-interfaces <IP>
nmap -sU -p 161 --script=snmp-brute <IP>
nmap -sV -sU -p 161 <IP>

# community string 테스트
snmpwalk -v 2c -c public <IP>
snmpwalk -v 2c -c private <IP>
snmpwalk -v 1 -c public <IP>
```

---

## 2. 기본 Community String

```
public   (읽기, 기본값)
private  (읽기/쓰기, 기본값)
manager
community
snmpd
```

---

## 3. snmpwalk — 정보 수집

```bash
# 전체 MIB 트리 수집
snmpwalk -v 2c -c public <IP>
snmpwalk -v 2c -c public <IP> > snmp_full.txt

# 시스템 정보
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1    # system
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.1.5  # sysName (호스트명)

# 네트워크 인터페이스
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.2    # interfaces

# 라우팅 테이블
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.4.21  # ipRouteTable

# ARP 테이블 (내부 IP 발견)
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.3.1   # atTable

# 실행 중인 프로세스
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.4.2   # hrSWRunName

# 설치된 소프트웨어
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.25.6.3   # hrSWInstalledName

# 열린 TCP 포트
snmpwalk -v 2c -c public <IP> 1.3.6.1.2.1.6.13     # tcpConnTable

# 사용자 계정 (Windows)
snmpwalk -v 2c -c public <IP> 1.3.6.1.4.1.77.1.2.25  # Windows users
```

---

## 4. onesixtyone — community string 브루트포스

```bash
# 기본 워드리스트
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt <IP>

# 서브넷 전체 스캔
onesixtyone -c community_strings.txt -i targets.txt
```

---

## 5. snmpset — 값 변경 (쓰기 권한)

```bash
# community string이 private이고 쓰기 권한 있을 때
snmpset -v 2c -c private <IP> 1.3.6.1.2.1.1.5.0 s "new_hostname"
```

---

## 6. snmp-check — 자동화 수집

```bash
snmp-check <IP>
snmp-check <IP> -c public -v 2c
```

---

## 유용한 OID 목록

```
1.3.6.1.2.1.1.1.0    sysDescr (OS 및 버전)
1.3.6.1.2.1.1.5.0    sysName (호스트명)
1.3.6.1.2.1.25.4.2   hrSWRunName (실행 중 프로세스)
1.3.6.1.2.1.25.6.3   hrSWInstalledName (설치된 소프트웨어)
1.3.6.1.4.1.77.1.2.25  Windows 사용자 목록
1.3.6.1.2.1.4.34.1.3   IPv6 주소
```

---

## 팁

- UDP 161이므로 nmap에 `sU` 필수
- SNMPv1/v2c는 community string이 평문 → 스니핑 가능
- SNMPv3는 인증/암호화 지원 → 브루트포스 필요
- 발견된 내부 IP/호스트 정보로 네트워크 맵 구성