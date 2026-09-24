# 09_EyeWitness_웹스크린샷

웹 vhost/서브도메인 수가 많을 때 하나씩 브라우저보는 대신 스크린샷을 한번에 뜨거워서 우선순위를 정하는 도구.

---

## EyeWitness

```bash
# 설치 (Kali에 기본 포함, 없으면)
sudo apt install eyewitness

# 해결한 서브도메인/vhost 목록 파일로 스크린샷
vi targets.txt   # 한 줄에 하나씩 (http:// 생략 가능)
eyewitness -f targets.txt -d OUTPUT_DIR

# nmap XML 결과로부터 바로 스크린샷 (대규모 내부망 스코프에 유리)
nmap -p80,443,8080,8443 -oX scan.xml <IP대역>
eyewitness -x scan.xml -d OUTPUT_DIR

# 타임아웃/스레드 조정 (VPN을 거치는 환경은 지연이 커서 기본값으로 타임아웃이 잘 날 수 있음)
eyewitness -f targets.txt -d OUTPUT_DIR --timeout 60

# 결과 보기
eyewitness -f targets.txt -d OUTPUT_DIR --no-prompt   # 종료 후 자동으로 브라우저 열기 안 물어보고 진행
firefox OUTPUT_DIR/report.html
```

리포트는 각 호스트별로 스크린샷 + 응답 헤더(서버, 버전, 쿠키, 타이틀)를 한눈에 보여주며, 기술스택이 교잡된 목록을 수십 개 이상 훑을 때 필수적인 도구다.

---

## 대안 도구 비교 (실전 벤치마크 기준)

| 도구 | 특징 | 평가 |
| --- | --- | --- |
| EyeWitness | Python, 서버 헤더·기본 로그인 탐지 기능까지 포함 | 설치/의존성 문제 종종 발생, 정확도 중간 |
| gowitness | Golang, Chrome Headless 기반 | 실측 정확도 98.7%로 최고, 설치/속도 다 좋음 — 현재 가장 추천 |
| WitnessMe | Python3, Pyppeteer 기반 | EyeWitness보다 설치 편함, CLI UX 좋음. 유사항목 그룹화 기능만 없음 |
| Aquatone | Golang(원래 Ruby) | 2019년 이후 유지보수 중단, 빌드 오류 많음 — 비추천 |

```bash
# gowitness 사용법 (추천 대안)
go install github.com/sensepost/gowitness@latest
gowitness scan file -f targets.txt
gowitness report server   # 웹 UI로 결과 확인
```

---

## 팁

- 보이지 않는 링크도 vhost 자체에 존재할 수 있으니, 스크린샷은 후보군 추릴 용도일 뿐, 디렉토리 브루트포스/vhost 퍼징은 별도로 진행해야 함
- 응답 헤더(서버 버전, X-Powered-By 등)만 봐도 기술스택(WordPress/Drupal/Flask 등)을 미리 판단해서 다음 단계 도구 선택(WPScan, droopescan 등)을 정할 수 있음
- 최근 버전 공식 문서는 HTTP 스크린샷에 `--web` 플래그를 명시적으로 붙이는 예제가 많음. 지금 확인된 설치 버전은 `--web` 없이도 정상 동작했지만, 다른 환경에서 "Attempting to screenshot"이 아예 안 뜨면 `--web` 붙여서 재시도해볼 것