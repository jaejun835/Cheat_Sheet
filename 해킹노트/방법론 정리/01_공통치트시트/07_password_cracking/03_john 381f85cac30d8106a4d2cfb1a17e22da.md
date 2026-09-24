# 03_john

# john (John the Ripper)

---

## 개념

John the Ripper(john)은 hashcat과 함께 가장 많이 쓰이는 패스워드 크래킹 툴이다. hashcat이 GPU 가속 기반으로 순수 해시 크래킹에 강하다면, john은 파일에서 해시를 추출하는 *2john 도구 생태계가 강점이다. 파일 형식별로 대응하는* 2john 도구가 있어 아카이브, 문서, 인증서 등 다양한 파일의 패스워드를 크래킹할 때 필수적이다.

Jumbo 버전(kali 기본 설치)이 훨씬 많은 포맷을 지원하므로 반드시 jumbo 버전을 사용해야 한다.

---

## 기본 사용법

```bash
# 자동 포맷 감지 + 크랙 (Single → Wordlist → Incremental 순)
john hash.txt

# 워드리스트 지정
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# 멀티코어 활용
john --wordlist=rockyou.txt --fork=4 hash.txt

# 크랙 재개 (중단된 경우)
john --restore

# 세션 이름 지정
john --session=mysession --wordlist=rockyou.txt hash.txt
john --restore=mysession    # 재개
```

---

## 포맷 지정 / 결과 확인

```bash
# 지원 포맷 전체 목록
john --list=formats
john --list=formats | grep -i bcrypt
john --list=formats | grep -i office
john --list=formats | grep -i sha

# 포맷 명시
john --format=NT hash.txt                 # NTLM
john --format=netntlmv2 hash.txt          # NetNTLMv2
john --format=krb5tgs hash.txt            # Kerberoast
john --format=krb5asrep hash.txt          # AS-REP Roast
john --format=bcrypt hash.txt             # bcrypt
john --format=sha512crypt hash.txt        # SHA512crypt $6$
john --format=sha256crypt hash.txt        # SHA256crypt $5$
john --format=md5crypt hash.txt           # MD5crypt $1$
john --format=raw-md5 hash.txt            # Raw MD5
john --format=raw-sha1 hash.txt           # Raw SHA1
john --format=raw-sha256 hash.txt         # Raw SHA256
john --format=phpass hash.txt             # WordPress/phpBB
john --format=pdf hash.txt                # PDF
john --format=zip hash.txt                # ZIP
john --format=rar hash.txt                # RAR
john --format=7z hash.txt                 # 7-Zip
john --format=gpg hash.txt                # GPG
john --format=ssh hash.txt                # SSH 개인키
john --format=passwordsafe hash.txt       # psafe3
john --format=keepass hash.txt            # KeePass

# 룰 적용
john --wordlist=rockyou.txt --rules hash.txt
john --wordlist=rockyou.txt --rules=best64 hash.txt
john --wordlist=rockyou.txt --rules=KoreLogic hash.txt
john --wordlist=rockyou.txt --rules=jumbo hash.txt

# 크랙된 결과 확인
john --show hash.txt
john --show --format=NT hash.txt

# 크랙 결과 저장 위치
cat ~/.john/john.pot
```

---

## 2john 계열 전체 목록

파일에서 john이 읽을 수 있는 해시 포맷으로 변환하는 도구들이다.

```bash
# 설치된 *2john 전체 목록 확인
ls /usr/share/john/*2john* 2>/dev/null
locate *2john* 2>/dev/null

# --- 인증 / 키 ---
ssh2john id_rsa > id_rsa.hash                         # SSH 개인키 (RSA/DSA/ECDSA/Ed25519)
pfx2john cert.pfx > pfx.hash                          # PFX / PKCS#12
gpg2john private.gpg > gpg.hash                       # GPG / PGP 개인키

# --- 아카이브 ---
zip2john secret.zip > zip.hash                        # ZIP
rar2john archive.rar > rar.hash                       # RAR
7z2john archive.7z > 7z.hash                          # 7-Zip

# --- 문서 ---
pdf2john document.pdf > pdf.hash                      # PDF
office2john document.docx > office.hash               # MS Office (Word/Excel/PPT/OneNote 등)

# --- 비밀번호 관리자 ---
keepass2john database.kdbx > keepass.hash             # KeePass 1.x/2.x
keepass2john -k keyfile.key database.kdbx > kp.hash   # 키파일 있을 때
# psafe3는 변환 없이 직접 사용 (해시타입별_크랙 참조)

# --- 운영체제 ---
unshadow /etc/passwd /etc/shadow > shadow.hash        # 리눅스 /etc/shadow
# Windows SAM → secretsdump / mimikatz로 추출 후 바로 사용

# --- 브라우저 ---
mozilla2john key4.db > mozilla.hash                   # Firefox / Thunderbird

# --- macOS ---
keychain2john login.keychain > keychain.hash          # macOS Keychain
dmg2john disk.dmg > dmg.hash                          # macOS .dmg

# --- 암호화폐 ---
bitcoin2john wallet.dat > bitcoin.hash                # Bitcoin Core

# --- Wifi ---
hccapx2john capture.hccapx > wifi.hash                # WPA/WPA2 핸드셰이크

# --- Kerberos ---
krb2john tickets.pcapng > krb.hash                    # pcap에서 Kerberos 티켓 추출

# --- 암호화 디스크 ---
luks2john /dev/sdb > luks.hash                        # LUKS 암호화 디스크
```