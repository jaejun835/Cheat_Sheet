# 01_mdb

# 파일 포맷 분석 — mdb (Microsoft Access Database)

---

## 개념

mdb는 Microsoft Access 데이터베이스 파일이다. JET 데이터베이스 엔진을 사용하며 Access 97~2003 버전은 `.mdb`, Access 2007 이상은 `.accdb` 확장자를 사용한다. 침투 테스트에서 FTP나 SMB 공유에서 발견되는 경우가 많으며, 유저 자격증명, 내부 시스템 정보, 설정값 등이 담겨있는 경우가 있다. 칼리에서 mdbtools로 Windows 없이 읽을 수 있다.

---

## 기본 분석

```bash
# 설치
sudo apt install mdbtools

# 파일 타입 확인 (확장자 믿지 말고 실제 타입 확인)
file database.mdb
# Microsoft Access Database (JET4) 또는 유사하게 출력됨

# 파일 버전 확인
# JET3 = Access 97
# JET4 = Access 2000 / 2002 / 2003
mdb-ver database.mdb

# 테이블 목록 확인 (공백 구분으로 출력)
mdb-tables database.mdb

# 테이블 목록을 한 줄씩 출력 (루프 스크립트에 사용)
mdb-tables -1 database.mdb

# 테이블 개수 확인
mdb-tables -1 database.mdb | wc -l

# 데이터베이스 스키마 확인 (테이블 구조, 컬럼명, 타입 파악)
# 어떤 컬럼이 있는지 미리 보고 탐색 방향 결정
mdb-schema database.mdb
```

---

## 테이블 탐색 전략

mdb 파일을 열었을 때 테이블이 많으면 어디를 봐야 할지 막힐 수 있다. 테이블 하나만 보고 끝내지 말고, 자격증명이 있을 법한 테이블을 전부 확인해야 한다. 같은 DB 안에서도 유저 정보, 인증 정보, 시스템 설정이 각각 다른 테이블에 분산되어 있는 경우가 많다.

```bash
# 1단계: 자격증명 관련 테이블명 먼저 grep
# 테이블명에 아래 키워드가 있으면 우선순위로 확인
mdb-tables -1 database.mdb | grep -i "user\|pass\|account\|login\|cred\|auth\|admin\|staff\|member\|employee"

# 2단계: 찾은 테이블 내용 전체 출력
mdb-export database.mdb USERINFO
mdb-export database.mdb auth_user
mdb-export database.mdb Accounts

# 3단계: 전체 테이블 내용 한번에 추출해서 키워드 grep
# 테이블명으로 못 찾았을 때 내용 기반으로 탐색
for table in $(mdb-tables -1 database.mdb); do
  echo "===$table ==="
  mdb-export database.mdb "$table" 2>/dev/null
  echo ""
done | grep -i "pass\|pwd\|secret\|key\|token\|admin\|credential"

# 4단계: 스키마에서 컬럼명 기반 탐색
# 컬럼명에 password, pwd 등이 있는 테이블 찾기
mdb-schema database.mdb | grep -i "password\|passwd\|pwd\|secret\|token"
```

---

## 데이터 추출

```bash
# 특정 테이블 전체 출력 (CSV 형식)
mdb-export database.mdb <테이블명>

# 파일로 저장
mdb-export database.mdb <테이블명> > output.csv

# JSON 형식으로 추출
mdb-json database.mdb <테이블명>

# 전체 테이블 각각 파일로 저장
mkdir mdb_dump
for table in $(mdb-tables -1 database.mdb); do
  mdb-export database.mdb "$table" > "mdb_dump/${table}.csv" 2>/dev/null
done
ls mdb_dump/
```

---

## SQL 직접 조회 (mdb-sql)

```bash
# 대화형 SQL 콘솔 실행
mdb-sql database.mdb

# 프롬프트에서 사용 가능한 명령어
# .tables                                 테이블 목록 확인
# select * from Users;                    테이블 전체 조회
# select username, password from auth_user where password != '';
# select count(*) from Users;             행 수 확인
# select * from Users limit 5;           상위 5개만
# .quit                                   종료
```

---

## 크레덴셜 재사용 패턴

mdb에서 발견한 패스워드는 다른 서비스에서도 재사용되는 경우가 매우 흔하다. 특히 암호화된 아카이브 파일(ZIP 등)이 같은 경로에 있으면 mdb에서 발견한 패스워드를 먼저 시도한다.

```
mdb 발견 패스워드
  → ZIP / RAR 암호 시도 (같은 경로에 있는 아카이브)
  → FTP / SSH / Telnet / SMB 로그인 시도
  → 웹 애플리케이션 로그인 시도
  → 도메인 계정 패스워드 스프레이
  → 발견한 유저명과 패스워드 조합 모두 시도
```