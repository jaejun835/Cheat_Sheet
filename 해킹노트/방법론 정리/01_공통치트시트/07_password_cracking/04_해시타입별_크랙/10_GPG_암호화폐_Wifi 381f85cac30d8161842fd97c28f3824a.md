# 10_GPG_암호화폐_Wifi

# 해시 타입별 크랙 — GPG / 암호화폐 / Wifi

---

## GPG / PGP 개인키

```
대상: 패스프레이즈로 보호된 GPG/PGP 개인키
방법: gpg2john으로 해시 추출 → john
      .gpg, .asc, .pgp 확장자 모두 지원
```

```bash
# 해시 추출
gpg2john private.gpg > gpg.hash
gpg2john private_key.asc > gpg.hash

# john으로 크랙
john --wordlist=rockyou.txt gpg.hash
john --format=gpg --wordlist=rockyou.txt gpg.hash
john --show gpg.hash
```

---

## Bitcoin wallet.dat

```
대상: Bitcoin Core wallet.dat 파일
방법: bitcoin2john으로 해시 추출 → john
```

```bash
# 해시 추출
bitcoin2john wallet.dat > bitcoin.hash

# john으로 크랙
john --wordlist=rockyou.txt bitcoin.hash
john --show bitcoin.hash
```

---

## Wifi WPA/WPA2

```
대상: WPA/WPA2 핸드셰이크 캡처 파일
방법: hcxpcapngtool (최신) 또는 hccapx2john (구버전) → hashcat / john
필요: 4-way 핸드셰이크 또는 PMKID 캡처 필요
```

```bash
# 최신 방식 — .pcapng → .hc22000 변환 후 hashcat
hcxpcapngtool -o hash.hc22000 capture.pcapng
hashcat -m 22000 hash.hc22000 rockyou.txt
hashcat -m 22000 hash.hc22000 rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# 구버전 .hccapx 방식
hashcat -m 2500 capture.hccapx rockyou.txt

# john 방식 (hccapx2john 사용)
hccapx2john capture.hccapx > wifi.hash
john --wordlist=rockyou.txt wifi.hash
john --show wifi.hash
```

---

## Cisco / 네트워크 장비

```
Cisco 패스워드는 타입 번호로 구분한다.
Type 7은 XOR 암호화라 복호화가 가능하다 (크래킹 불필요).
```

```bash
# Cisco Type 5 ($1$, MD5crypt) — mode 500
# 형식: $1$<salt>$<hash>
hashcat -m 500 hash.txt rockyou.txt
john --format=md5crypt --wordlist=rockyou.txt hash.txt

# Cisco Type 7 — XOR 암호화 (복호화 가능, 크래킹 불필요)
# 온라인 복호화: https://www.ifm.net.nz/cookbooks/passwordcracker.html
# python으로 복호화:
python3 -c "
key = [0x64,0x73,0x66,0x64,0x3b,0x6b,0x66,0x6f,0x41,0x2c,0x2e,0x69,0x79,0x65,0x77,0x72,0x6b,0x6c,0x64,0x4a,0x4b,0x44,0x48,0x53,0x55,0x42]
enc = '0822455D0A16'   # 여기에 암호화된 값 입력
s = int(enc[:2])
result = ''
for i, c in enumerate(enc[2:][::2]):
    result += chr(int(enc[2:][i*2:i*2+2], 16) ^ key[(s+i) % 26])
print(result)
"

# Cisco Type 8 ($8$, PBKDF2-SHA256) — mode 9200
hashcat -m 9200 hash.txt rockyou.txt

# Cisco Type 9 ($9$, scrypt) — mode 9300 (매우 느림)
hashcat -m 9300 hash.txt rockyou.txt
john --format=scrypt --wordlist=rockyou.txt hash.txt
```