# 서비스별 NSE 스크립트

각 서비스는 **기본 열거 → 상세 열거 → 특정 취약점 검사 → 필요할 때만 인증 시도** 순서로 진행한다.

### FTP — 21/TCP

FTP에서는 익명 로그인, 서버 정보, 지원 기능과 파일 목록을 먼저 확인한다.

```bash
# 기본 열거
sudo nmap -Pn -sV -p21 \
  --script="ftp-anon,ftp-syst,banner" \
  <IP> \
  -oA ftp_basic

# 상세 열거
sudo nmap -Pn -sV --version-all -p21 \
  --script="ftp-anon,ftp-syst,ftp-bounce,banner" \
  <IP> \
  -oA ftp_detail

# 익명 로그인 시 최대 100개 항목 표시
sudo nmap -Pn -p21 \
  --script=ftp-anon \
  --script-args='ftp-anon.maxlist=100' \
  <IP>

# vsftpd 2.3.4 백도어가 의심될 때만
sudo nmap -Pn -sV -p21 \
  --script=ftp-vsftpd-backdoor \
  <IP>

# ProFTPD CVE-2010-4221 대상 버전일 때만
sudo nmap -Pn -sV -p21 \
  --script=ftp-vuln-cve2010-4221 \
  <IP>

# 브루트포스
sudo nmap -Pn -p21 \
  --script=ftp-brute \
  --script-args='userdb=users.txt,passdb=pass.txt,brute.firstonly=true' \
  <IP>
```

---

### SSH — 22/TCP

SSH에서는 버전, 호스트키, 인증 방식과 지원 알고리즘을 확인한다. 오래된 버전이 보여도 Ubuntu 백포트 패치 여부를 확인해야 한다.

```bash
# 기본 열거
sudo nmap -Pn -sV -p22 \
  --script="ssh-hostkey,ssh-auth-methods" \
  <IP> \
  -oA ssh_basic

# 상세 열거
sudo nmap -Pn -sV --version-all -p22 \
  --script="ssh-hostkey,ssh-auth-methods,ssh2-enum-algos" \
  <IP> \
  -oA ssh_detail

# 특정 사용자 기준 인증 방식 확인
sudo nmap -Pn -p22 \
  --script=ssh-auth-methods \
  --script-args='ssh.user=admin' \
  <IP>

# 브루트포스
sudo nmap -Pn -p22 \
  --script=ssh-brute \
  --script-args='userdb=users.txt,passdb=pass.txt,brute.firstonly=true' \
  <IP>
```

---

### SMTP — 25, 465, 587/TCP

SMTP에서는 지원 명령, 사용자 열거 가능성, NTLM 정보와 오픈 릴레이 여부를 확인한다.

```bash
# 기본 열거
sudo nmap -Pn -sV -p25,465,587 \
  --script="smtp-commands,smtp-ntlm-info" \
  <IP> \
  -oA smtp_basic

# 상세 열거
sudo nmap -Pn -sV -p25,465,587 \
  --script="smtp-commands,smtp-enum-users,smtp-open-relay,smtp-ntlm-info,ssl-cert,ssl-enum-ciphers" \
  <IP> \
  -oA smtp_detail

# VRFY 방식으로 사용자 열거
sudo nmap -Pn -p25 \
  --script=smtp-enum-users \
  --script-args='smtp-enum-users.methods=VRFY' \
  <IP>

# 여러 사용자 열거 방식 사용
sudo nmap -Pn -p25 \
  --script=smtp-enum-users \
  --script-args='smtp-enum-users.methods={VRFY,EXPN,RCPT}' \
  <IP>

# 특정 Exim 버전이 의심될 때만
sudo nmap -Pn -sV -p25 \
  --script=smtp-vuln-cve2010-4344 \
  <IP>
```

---

### DNS — 53/TCP·UDP

DNS는 UDP 53과 TCP 53을 모두 확인한다. 재귀 질의, 영역 전송, 하위 도메인과 서비스 레코드를 열거한다.

```bash
# TCP와 UDP 53번 포트 확인
sudo nmap -Pn -sS -sU \
  -pT:53,U:53 \
  -sV <IP> \
  -oA dns_ports

# 기본 열거
sudo nmap -Pn -sU -sV -p53 \
  --script="dns-recursion,dns-service-discovery" \
  <IP> \
  -oA dns_basic

# 도메인을 알고 있을 때 상세 열거
sudo nmap -Pn -sU -p53 \
  --script="dns-recursion,dns-brute,dns-service-discovery" \
  --script-args='dns-brute.domain=corp.local' \
  <IP> \
  -oA dns_detail

# 영역 전송은 TCP로 확인
sudo nmap -Pn -sT -p53 \
  --script=dns-zone-transfer \
  --script-args='dns-zone-transfer.domain=corp.local' \
  <IP>
```

---

### HTTP/HTTPS — 80, 443, 8080, 8443/TCP

웹 서비스에서는 제목, 헤더, 메서드, 숨겨진 경로, robots.txt, 인증 페이지와 백업 파일을 확인한다.

```bash
# 기본 웹 열거
sudo nmap -Pn -sV -p80,443,8080,8443 \
  --script="http-title,http-server-header,http-headers,http-methods,http-enum" \
  <IP> \
  -oA http_basic

# 상세 웹 열거
sudo nmap -Pn -sV --version-all -p80,443,8080,8443 \
  --script="http-title,http-server-header,http-headers,http-methods,http-enum,http-robots.txt,http-generator,http-auth-finder,http-default-accounts,http-config-backup" \
  <IP> \
  -oA http_detail

# HTTPS 인증서와 지원 암호군 확인
sudo nmap -Pn -sV -p443,8443 \
  --script="http-title,http-headers,http-methods,http-enum,ssl-cert,ssl-enum-ciphers" \
  <IP> \
  -oA https_detail

# robots.txt 확인
sudo nmap -Pn -p80,443 \
  --script=http-robots.txt \
  <IP>

# PHP 버전 단서 확인
sudo nmap -Pn -p80,443 \
  --script=http-php-version \
  <IP>

# WordPress 흔적 열거
sudo nmap -Pn -p80,443 \
  --script=http-wordpress-enum \
  <IP>

# 노출된 Git 저장소 확인
sudo nmap -Pn -p80,443 \
  --script=http-git \
  <IP>

# CGI 경로와 Shellshock 조건이 확인됐을 때만
sudo nmap -Pn -p80 \
  --script=http-shellshock \
  --script-args='http-shellshock.uri=/cgi-bin/test.sh' \
  <IP>

# Apache Struts 대상임이 확인됐을 때만
sudo nmap -Pn -p80,443 \
  --script=http-vuln-cve2017-5638 \
  <IP>

# 웹 취약점 스크립트 후보 검사
# 결과를 확정 취약점으로 간주하지 않음
sudo nmap -Pn -sV -p80,443,8080,8443 \
  --script="http-vuln-*" \
  <IP> \
  -oA http_vuln_candidates
```

Nmap만으로 웹 애플리케이션을 전부 분석할 수는 없다. 함께 사용할 도구:

```bash
whatweb http://<IP>
gobuster dir -u http://<IP>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
nikto -h http://<IP>
wpscan --url http://<IP>
```

---

### SMB — 139, 445/TCP

SMB에서는 호스트명, OS 단서, SMB 버전, signing 설정, 공유, 사용자와 그룹을 확인한다.

```bash
# 기본 SMB 열거
sudo nmap -Pn -sV -p139,445 \
  --script="nbstat,smb-os-discovery,smb-protocols,smb-security-mode,smb2-security-mode,smb-enum-shares" \
  <IP> \
  -oA smb_basic

# 상세 SMB 열거
sudo nmap -Pn -sV --version-all -p139,445 \
  --script="nbstat,smb-os-discovery,smb-protocols,smb-security-mode,smb2-security-mode,smb-enum-shares,smb-enum-users,smb-enum-groups,smb-enum-services,smb-system-info" \
  <IP> \
  -oA smb_detail

# SMB signing만 확인
sudo nmap -Pn -p139,445 \
  --script="smb-security-mode,smb2-security-mode" \
  <IP>

# MS17-010 전용 검사
# SMBv1 Windows 대상인지 먼저 확인
sudo nmap -Pn -p445 \
  --script=smb-vuln-ms17-010 \
  <IP>

# MS08-067 전용 검사
# 오래된 Windows 대상에서만 의미 있음
sudo nmap -Pn -p445 \
  --script=smb-vuln-ms08-067 \
  <IP>

# CVE-2009-3103 전용 검사
sudo nmap -Pn -p445 \
  --script=smb-vuln-cve2009-3103 \
  <IP>

# SMB 취약점 스크립트 전체 실행
# 대상과 맞지 않는 오래된 검사와 오탐이 포함될 수 있어 기본 사용 비권장
sudo nmap -Pn -p139,445 \
  --script="smb-vuln-*" \
  <IP> \
  -oA smb_vuln_candidates
```

SMB는 다음 도구와 교차 확인한다.

```bash
nxc smb <IP>
smbclient -N -L //<IP>
enum4linux-ng -A <IP>
```

---

### LDAP — 389, 636/TCP

LDAP에서는 RootDSE, naming context, 도메인 정보와 익명 검색 가능 여부를 확인한다.

```bash
# 기본 LDAP 열거
sudo nmap -Pn -sV -p389,636 \
  --script="ldap-rootdse,ssl-cert" \
  <IP> \
  -oA ldap_basic

# 상세 LDAP 열거
sudo nmap -Pn -sV -p389,636 \
  --script="ldap-rootdse,ldap-search,ssl-cert,ssl-enum-ciphers" \
  <IP> \
  -oA ldap_detail

# Base DN을 알고 있을 때
sudo nmap -Pn -p389 \
  --script=ldap-search \
  --script-args='ldap.base="DC=corp,DC=local"' \
  <IP>
```

---

### MySQL — 3306/TCP

MySQL에서는 버전, 인증 플러그인, 빈 비밀번호, 데이터베이스와 사용자 정보를 확인한다.

```bash
# 기본 MySQL 열거
sudo nmap -Pn -sV -p3306 \
  --script="mysql-info,mysql-empty-password" \
  <IP> \
  -oA mysql_basic

# 상세 MySQL 열거
sudo nmap -Pn -sV -p3306 \
  --script="mysql-info,mysql-empty-password,mysql-databases,mysql-users,mysql-variables" \
  <IP> \
  -oA mysql_detail

# 자격 증명을 알고 있을 때
sudo nmap -Pn -p3306 \
  --script="mysql-databases,mysql-users,mysql-variables" \
  --script-args='mysqluser=root,mysqlpass=Password123' \
  <IP>

# 브루트포스
sudo nmap -Pn -p3306 \
  --script=mysql-brute \
  --script-args='userdb=users.txt,passdb=pass.txt,brute.firstonly=true' \
  <IP>
```

---

### MSSQL — 1433/TCP

MSSQL에서는 인스턴스 정보, NTLM 정보, 설정, 빈 비밀번호와 데이터베이스 접근 권한을 확인한다.

```bash
# 기본 MSSQL 열거
sudo nmap -Pn -sV -p1433 \
  --script="ms-sql-info,ms-sql-ntlm-info,ms-sql-empty-password" \
  <IP> \
  -oA mssql_basic

# 상세 MSSQL 열거
sudo nmap -Pn -sV -p1433 \
  --script="ms-sql-info,ms-sql-ntlm-info,ms-sql-config,ms-sql-empty-password,ms-sql-dac,ms-sql-hasdbaccess" \
  <IP> \
  -oA mssql_detail

# 자격 증명을 알고 있을 때
sudo nmap -Pn -p1433 \
  --script="ms-sql-config,ms-sql-hasdbaccess" \
  --script-args='mssql.username=sa,mssql.password=Password123' \
  <IP>

# xp_cmdshell 명령 실행
# 유효한 자격 증명과 명시적 허가가 있을 때만
sudo nmap -Pn -p1433 \
  --script=ms-sql-xp-cmdshell \
  --script-args='mssql.username=sa,mssql.password=Password123,ms-sql-xp-cmdshell.cmd=whoami' \
  <IP>
```

---

### RDP — 3389/TCP

RDP에서는 암호화 수준, NLA, NTLM 정보와 인증서를 확인한다.

```bash
# 기본 RDP 열거
sudo nmap -Pn -sV -p3389 \
  --script="rdp-enum-encryption,rdp-ntlm-info" \
  <IP> \
  -oA rdp_basic

# 상세 RDP 열거
sudo nmap -Pn -sV -p3389 \
  --script="rdp-enum-encryption,rdp-ntlm-info,ssl-cert,ssl-enum-ciphers" \
  <IP> \
  -oA rdp_detail

# MS12-020 검사
# BlueKeep 검사가 아니며 오래된 Windows RDP 대상에만 의미 있음
sudo nmap -Pn -p3389 \
  --script=rdp-vuln-ms12-020 \
  <IP>
```

---

### SNMP — 161/UDP

SNMP에서는 커뮤니티 문자열, 시스템 정보, 인터페이스, 프로세스, 라우팅과 설치 소프트웨어를 확인한다.

```bash
# 기본 SNMP 열거
sudo nmap -Pn -sU -sV -p161 \
  --script="snmp-info,snmp-sysdescr" \
  <IP> \
  -oA snmp_basic

# public 커뮤니티 문자열로 상세 열거
sudo nmap -Pn -sU -sV -p161 \
  --script="snmp-info,snmp-sysdescr,snmp-interfaces,snmp-netstat,snmp-processes" \
  --script-args='snmpcommunity=public' \
  <IP> \
  -oA snmp_detail

# Windows SNMP 대상일 때 추가
sudo nmap -Pn -sU -sV -p161 \
  --script="snmp-win32-services,snmp-win32-software,snmp-win32-users" \
  --script-args='snmpcommunity=public' \
  <IP>

# 커뮤니티 문자열 브루트포스
sudo nmap -Pn -sU -p161 \
  --script=snmp-brute \
  <IP>

# 커뮤니티 목록 지정
sudo nmap -Pn -sU -p161 \
  --script=snmp-brute \
  --script-args='snmp-brute.communitiesdb=communities.txt' \
  <IP>
```

---