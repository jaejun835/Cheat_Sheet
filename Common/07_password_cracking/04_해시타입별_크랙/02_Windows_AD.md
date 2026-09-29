# 02_Windows_AD

# 해시 타입별 크랙 — Windows / AD

---

## 개념

Windows/AD 환경에서 획득하는 해시 종류는 여러 가지다. 각각 획득 경로와 크래킹 방식이 다르다. NTLM 해시는 SAM 덤프/secretsdump/mimikatz로 획득하고, NetNTLMv2는 Responder로 네트워크에서 캡처한다. Kerberos 해시는 Kerberoasting/AS-REP Roasting으로 획득한다.

---

## NTLM (mode 1000)

```
형식: 32자 hex
획득: secretsdump, mimikatz, SAM 덤프
secretsdump 출력: Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NTLM>:::
                                                                        ^^^^ 이 부분
```

```bash
# secretsdump 출력에서 NTLM만 추출
cat secretsdump.txt | cut -d: -f4 > ntlm.hash

# hashcat (매우 빠름 — GPU로 수십 GH/s)
hashcat -m 1000 ntlm.hash rockyou.txt
hashcat -m 1000 ntlm.hash rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 1000 ntlm.hash rockyou.txt -r /usr/share/hashcat/rules/d3ad0ne.rule

# john
john --format=NT --wordlist=rockyou.txt ntlm.hash

# 결과 확인
hashcat -m 1000 ntlm.hash --show
john --show --format=NT ntlm.hash
```

---

## NetNTLMv1 (mode 5500)

```
형식: user::domain:LM_response:NTLM_response:challenge
획득: Responder (LLMNR/NBT-NS 포이즈닝)
특징: NTLMv2보다 취약, LM response로 다운그레이드 가능
```

```bash
# Responder 캡처 파일에서 가져오기
cat /usr/share/responder/logs/SMB-NTLMv1-SSP-*.txt > ntlmv1.hash

hashcat -m 5500 ntlmv1.hash rockyou.txt
hashcat -m 5500 ntlmv1.hash rockyou.txt -r /usr/share/hashcat/rules/best64.rule

john --format=netntlm --wordlist=rockyou.txt ntlmv1.hash
```

---

## NetNTLMv2 (mode 5600)

```
형식: user::domain:challenge:HMAC-MD5:blob
획득: Responder (LLMNR/NBT-NS 포이즈닝)
특징: 현대 환경에서 가장 흔히 캡처되는 형태
주의: PtH 불가 (해시 자체가 인증 증명이 아님), 크래킹 또는 릴레이만 가능
```

```bash
# Responder 캡처 파일
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt > ntlmv2.hash

hashcat -m 5600 ntlmv2.hash rockyou.txt
hashcat -m 5600 ntlmv2.hash rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 5600 ntlmv2.hash rockyou.txt -r /usr/share/hashcat/rules/OneRuleToRuleThemAll.rule

john --format=netntlmv2 --wordlist=rockyou.txt ntlmv2.hash

hashcat -m 5600 ntlmv2.hash --show
```

---

## Kerberoast — TGS-REP (mode 13100 / 19600 / 19700)

```
형식: $krb5tgs$<타입>$*유저$도메인$SPN*$해시
획득: GetUserSPNs, nxc --kerberoasting, Rubeus kerberoast
타입별 모드:
  $23$ → RC4    → mode 13100  (가장 빠름, 크래킹 우선)
  $17$ → AES128 → mode 19600
  $18$ → AES256 → mode 19700  (가장 느림)
```

```bash
# RC4 크랙 — mode 13100 (초당 수백 MH/s)
hashcat -m 13100 kerb.hash rockyou.txt
hashcat -m 13100 kerb.hash rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 13100 kerb.hash rockyou.txt -r /usr/share/hashcat/rules/d3ad0ne.rule

# AES128
hashcat -m 19600 kerb.hash rockyou.txt

# AES256
hashcat -m 19700 kerb.hash rockyou.txt

# john
john --format=krb5tgs --wordlist=rockyou.txt kerb.hash

hashcat -m 13100 kerb.hash --show
```

---

## AS-REP Roast — AS-REP (mode 18200)

```
형식: $krb5asrep$<타입>$유저@도메인:해시
획득: GetNPUsers, nxc --asreproast, kerbrute (DONT_REQ_PREAUTH 계정)
타입별 모드:
  $23$ → RC4    → mode 18200
  $17$ → AES128 → john --format=krb5asrep
  $18$ → AES256 → john --format=krb5asrep
```

```bash
hashcat -m 18200 asrep.hash rockyou.txt
hashcat -m 18200 asrep.hash rockyou.txt -r /usr/share/hashcat/rules/best64.rule

john --format=krb5asrep --wordlist=rockyou.txt asrep.hash

hashcat -m 18200 asrep.hash --show
```

---

## Domain Cached Credentials (DCC / MS-Cache)

```
형식: $DCC$10240#username#hash
획득: mimikatz lsadump::cache, secretsdump
용도: 도메인 연결 없을 때 로컬 로그인에 사용되는 캐시된 자격증명
특징: PtH 불가, 크래킹만 가능. DCC2(Vista+)는 매우 느림
```

```bash
# DCC v1 — XP/2003 (mode 1100, 빠름)
hashcat -m 1100 dcc.hash rockyou.txt

# DCC v2 — Vista+ (mode 2100, 매우 느림 — PBKDF2 기반)
hashcat -m 2100 dcc.hash rockyou.txt
hashcat -m 2100 dcc.hash /usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt

john --format=mscash2 --wordlist=rockyou.txt dcc.hash
```