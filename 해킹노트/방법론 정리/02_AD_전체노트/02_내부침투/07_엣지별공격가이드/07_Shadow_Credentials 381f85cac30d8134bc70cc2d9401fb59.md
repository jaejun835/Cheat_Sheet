# 07_Shadow_Credentials

msDS-KeyCredentialLink 속성에 공격자 인증서를 등록해 PKINIT으로 인증한다. GenericAll 또는 GenericWrite가 있을 때도 동일한 공격이 가능하다. DC가 Windows Server 2016 이상이고 PKINIT을 지원해야 동작한다.

**GenericAll — 유저 대상 / Shadow Credentials 섹션** 동일

컴퓨터 대상은 `account` 값을 `<대상컴퓨터>$` 형태로 지정