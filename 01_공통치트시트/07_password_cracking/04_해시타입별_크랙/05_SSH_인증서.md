# 05_SSH_인증서

# 해시 타입별 크랙 — SSH / 인증서

---

## SSH 개인키 패스프레이즈

```
대상: 패스프레이즈로 보호된 SSH 개인키 (id_rsa, id_ecdsa, id_ed25519 등)
방법: ssh2john으로 해시 추출 → john으로 크랙
      hashcat은 SSH 개인키 직접 미지원 → john 사용
```

```bash
# 해시 추출
ssh2john id_rsa > id_rsa.hash
ssh2john id_ecdsa > ecdsa.hash
ssh2john id_ed25519 > ed25519.hash

# john으로 크랙
john --wordlist=rockyou.txt id_rsa.hash
john --format=ssh --wordlist=rockyou.txt id_rsa.hash

# 룰 적용
john --wordlist=rockyou.txt --rules id_rsa.hash

# 결과 확인
john --show id_rsa.hash
```

---

## PFX / PKCS#12 (.pfx / .p12)

```
대상: 개인키 + 인증서가 묶인 파일, 패스프레이즈로 보호
용도: Windows 인증서 내보내기, AD CS 인증서 등
방법: pfx2john으로 해시 추출 → john으로 크랙
```

```bash
# 해시 추출
pfx2john cert.pfx > pfx.hash
pfx2john cert.p12 > pfx.hash

# john으로 크랙
john --wordlist=rockyou.txt pfx.hash

# 결과 확인
john --show pfx.hash

# 크랙 성공 후 PFX 열기
openssl pkcs12 -in cert.pfx -out cert.pem -nodes -password pass:<패스워드>
```