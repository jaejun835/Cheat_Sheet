# 09_macOS

# 해시 타입별 크랙 — macOS

---

## macOS Keychain

```
대상: macOS 로그인 키체인 (login.keychain)
방법: keychain2john으로 해시 추출 → john
경로: ~/Library/Keychains/login.keychain-db (Keychain 2) 또는 login.keychain
```

```bash
# 해시 추출
keychain2john login.keychain > keychain.hash
keychain2john login.keychain-db > keychain.hash

# john으로 크랙
john --wordlist=rockyou.txt keychain.hash
john --show keychain.hash
```

---

## macOS .dmg

```
대상: 패스워드로 암호화된 macOS 디스크 이미지
방법: dmg2john으로 해시 추출 → john
```

```bash
# 해시 추출
dmg2john disk.dmg > dmg.hash

# john으로 크랙
john --wordlist=rockyou.txt dmg.hash
john --show dmg.hash
```