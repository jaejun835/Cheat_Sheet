# 05_eml

# 파일 포맷 분석 — eml (이메일 파일)

---

## 개념

eml은 RFC 822 표준 이메일 형식이다. 헤더에서 발신자 IP, 경유 서버, 이메일 클라이언트 정보를 확인할 수 있고, 첨부파일을 추출할 수 있다. PST에서 개별 추출하거나 웹 서버에서 발견되는 경우가 있다.

---

## 기본 분석

```bash
# 파일 타입 확인
file email.eml

# 전체 내용 확인 (헤더 + 본문 + 첨부파일 인코딩 포함)
cat email.eml | less

# 헤더만 확인 (발신자 IP, 경유 서버, 클라이언트 정보)
# Received: 헤더는 여러 개 있으며 이메일 경로를 역순으로 추적 가능
cat email.eml | grep -i "^received:\|^from:\|^to:\|^subject:\|^date:\|^x-mailer:\|^user-agent:"

# 발신 IP 추출
cat email.eml | grep "^Received:" | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"

# exiftool로 메타데이터 추출
exiftool email.eml
```

---

## 본문 탐색

```bash
# 본문에서 자격증명 키워드 탐색
cat email.eml | grep -i "password\|passwd\|pass:\|pwd\|secret\|credential\|token\|api.key"

# 내부 URL, IP 추출
cat email.eml | grep -oE "https?://[a-zA-Z0-9./_?=-]+"
cat email.eml | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"

# Base64 인코딩된 본문 디코딩 (Content-Transfer-Encoding: base64)
cat email.eml | grep -A 100 "Content-Transfer-Encoding: base64" | \
  grep -v "^Content\|^--\|^$" | head -50 | base64 -d 2>/dev/null
```

---

## 첨부파일 추출

```bash
# munpack으로 첨부파일 추출 (가장 간단)
sudo apt install mpack
munpack email.eml
# 현재 디렉토리에 첨부파일이 저장됨

# Python으로 첨부파일 추출
python3 << 'EOF'
import email
import os

with open('email.eml', 'rb') as f:
    msg = email.message_from_bytes(f.read())

print(f"Subject: {msg['Subject']}")
print(f"From: {msg['From']}")
print(f"To: {msg['To']}")
print()

for part in msg.walk():
    if part.get_content_maintype() == 'multipart':
        continue
    filename = part.get_filename()
    content_type = part.get_content_type()
    print(f"파트: {content_type} / 파일명: {filename}")
    if filename:
        with open(filename, 'wb') as f:
            f.write(part.get_payload(decode=True))
        print(f"  → 추출됨: {filename}")
    elif content_type == 'text/plain':
        print("  [본문]")
        print(part.get_payload(decode=True).decode('utf-8', errors='ignore'))
EOF

# 추출된 첨부파일 확인
ls -la
file *  # 타입 확인
```