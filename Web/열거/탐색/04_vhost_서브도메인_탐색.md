# 04_vhost_서브도메인_탐색

# vhost / 서브도메인 탐색

하나의 IP 에 여러 사이트가 Host 헤더로 구분돼 올라가 있다. 공개 안 된 개발/관리 vhost 에서 취약점이 제일 많이 나온다. 디렉터리 탐색은 03 참고.

---

## 개념 - DNS 서브도메인 vs vhost

둘 다 "숨은 이름"을 찾지만 층위가 다르다.

- DNS 서브도메인 - 그 이름이 DNS 에 공식 등록돼 있는가. dig/dnsrecon/gobuster dns 로 찾는다
- vhost 퍼징 - DNS 에 있든 없든, 웹서버가 그 Host 헤더값을 알아듣고 다른 콘텐츠를 주는가. gobuster vhost/ffuf 로 찾는다

관리자가 DNS 엔 안 올리고 /etc/hosts 로만 쓰는 숨은 vhost 가 핵심이다. 그래서 DNS 열거를 먼저 하고, vhost 퍼징을 반드시 병행한다.

---

## DNS 서브도메인 탐색

### gobuster dns

```bash
gobuster dns -d <domain> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50 -o gb-dns.txt

# 특정 네임서버로 질의 (타겟 DNS 직접 지정)
gobuster dns -d <domain> -r <IP>:53 -w <wordlist> -t 50
```

### dig (수동 / 단건)

```bash
dig +short <sub>.<domain> @<nameserver>   # A 레코드
dig -x <IP> +short                        # PTR (역방향)
dig mx <domain> +short                     # 메일 서버
dig ns <domain> +short                     # 네임서버
```

### dnsrecon

```bash
dnsrecon -d <domain> -D <wordlist> -t brt -n <nameserver>
```

트러블슈팅 - 파이썬 3.12+ 에서 `FancyURLopener` 에러로 죽으면 (Bing/Yandex 모듈이 삭제된 기능 참조). brt 엔 안 쓰이니 그 줄만 무력화:

```bash
sudo sed -i 's/^url_opener = urllib.request.FancyURLopener/url_opener = object/' /usr/share/dnsrecon/dnsrecon/lib/*.py
```

---

## 와일드카드 함정 (중요)

DNS/vhost 둘 다, 서버가 "아무 이름에나 응답"하는 경우가 있다. 이러면 사전의 모든 단어가 "존재"로 뜨는 가짜 결과가 나온다 (localhost 까지 떴다면 와일드카드다).

```bash
# 먼저 존재할 리 없는 랜덤 이름을 질의해 와일드카드 응답 IP 를 확인
dig +short asdqwe12345notreal.<domain> @<nameserver>

# 그 IP 와 다른 응답만 진짜로 거른다 (export 로 서브셸에 전달)
export WILD=$(dig +short asdqwe12345notreal.<domain> @<nameserver> | head -1)
cat <wordlist> | xargs -P50 -I{} sh -c \
  'r=$(dig +short {}.<domain> @<nameserver> | head -1); [ -n "$r" ] && [ "$r" != "$WILD" ] && echo "[+] {} -> $r"'
```

- gobuster dns 는 `--wildcard` 로 강제 진행은 되지만, 걸러주진 않는다. 위 dig 방식이 확실하다
- dnsrecon 은 와일드카드를 자동 감지해 걸러준다

---

## vhost 퍼징 (Host 헤더)

### 준비 - 두 가지를 혼동 말 것

- /etc/hosts 등록 - 내 머신이 그 도메인을 IP 로 해석하게 (접속 자체를 가능하게)
- `--append-domain` - gobuster 가 사전 단어에 base 도메인을 붙여 완전한 Host 이름을 만들게 (요청 조립)

```bash
echo "<IP> <domain>" | sudo tee -a /etc/hosts
```

### gobuster vhost

```bash
gobuster vhost -u http://<domain>:<port>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 50 -o gb-vhost.txt
```

- `--append-domain` 필수. 빼면 Host 에 `web` 만 들어가 서버가 못 알아듣고 Found 0 이 된다 (gobuster 가 경고까지 띄움)
- URL 은 IP 말고 도메인으로 줘야 base 가 잡힌다. 포트가 기본이 아니면 `:port` 포함

### 결과 판독

- Status 200 (또는 301/302/403 등 정상 응답) → 진짜 vhost. 이것만 본다
- Status 400 → 사전 단어에 점(.)이 있어 `word..domain` 이 된 깨진 요청. vhost 없음이 아니라 형식 오류이므로 무시
- 모든 응답 크기가 같으면(와일드카드성) → `--exclude-length <size>` 로 기본 응답 걸러냄

```bash
# 400 노이즈 제거해서 보기
grep -v "Status: 400" gb-vhost.txt
```

### 사전 크기 상향

작은 사전에 답 vhost 가 없으면 단계적으로 키운다.

```
subdomains-top1million-5000.txt → -20000.txt → -110000.txt
```

### ffuf vhost (Host 헤더 직접 퍼징)

vhost 는 보통 전부 200 을 주므로, 가짜 응답 크기를 먼저 보고 `-fs` 로 거른다.

```bash
ffuf -u http://<IP>:<port>/ -H "Host: FUZZ.<domain>" -w <wordlist> -fs <default_size>
```

---

## 팁

- DNS 열거(gobuster dns/dnsrecon)와 vhost 퍼징은 찾는 게 다르다. 반드시 둘 다 한다
- vhost 는 `--append-domain` 없으면 아무것도 안 나온다. 1순위 체크 포인트
- 결과는 Status 200 만 진짜, 400 은 버린다
- 모든 게 참으로 뜨면 와일드카드를 의심하고, 랜덤 이름 응답과 다른 것만 거른다
- 찾은 vhost 는 /etc/hosts 에 등록해야 브라우저로 직접 열어볼 수 있다