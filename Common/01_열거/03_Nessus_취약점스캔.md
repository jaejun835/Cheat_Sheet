# 03_Nessus_취약점스캔

# Nessus (취약점 스캐너)

알려진 CVE, 패치 누락, 설정 오류를 플러그인으로 자동 점검하는 상용 취약점 스캐너. 넓은 범위를 빠르게 훑어 "뭐가 취약한지" 목록을 뽑는다. 실무 VA(취약점 진단)와 CPTS 의 스캐너 활용에 쓴다. 공격 중심 자격증(OSCP)이나 HTB 박스 풀이에선 보통 안 쓴다 (수동 열거가 핵심이라). 웹 UI 도구라 명령어가 아니라 작업 순서가 중심이다.

---

## 1. 접속

```
https://<IP>:8834
```

- HTTPS 전용 포트 8834. 자가서명 인증서라 브라우저 경고는 Advanced → Proceed
- 로그인 후 My Scans 에서 스캔 관리

---

## 2. 스캔 생성

```
New Scan → Basic Network Scan 선택
```

- **Settings 탭**
  - Name - 스캔 이름
  - Targets - 대상. IP, 범위(172.16.16.1-100), CIDR(172.16.16.0/24), 호스트명 모두 가능
- **모든 포트 스캔 (ALL ports)**
  - Discovery → Port Scanning → Port scan range 를 `all` (또는 `1-65535`) 로
  - 기본은 상위 일부 포트만 본다. 전체 포트를 봐야 할 땐 반드시 바꾼다
- **인증 스캔 (Credentials) - 중요**
  - Credentials 탭에서 계정 추가
    - Windows - SMB 계정 (administrator 등)
    - SSH - Linux 계정
  - 인증 스캔은 호스트 내부(설치된 패치, 로컬 설정, 공유 목록)까지 깊게 본다. 비인증보다 훨씬 많이 찾는다
- Save → Launch (▶) 로 실행. 전체 포트+인증 스캔은 수십 분~1시간 걸린다

---

## 3. 결과 판독

```
My Scans → 스캔 클릭 → Hosts → 호스트 클릭 → Vulnerabilities
```

- **Severity** - Critical / High / Medium / Low / Info. 목록은 severity+score 순 정렬이라 맨 위가 최상위
- **CVSS Score** - 10.0 이 최고. 같은 10.0 이 여러 개면 맨 위 항목이 "the highest"
- **Plugin ID** - 취약점 하나를 클릭 → 우측 Plugin Details 의 `ID`
- **Output** - 각 취약점 상세의 Output 에 근거가 나온다
  - Port - 어느 포트에서 탐지됐나 (예: VNC, SMB 포트 확인)
  - Path - 취약 파일 경로 (예: log4j jar 위치)
  - SMB Shares - 공유 목록과 접근 가능 여부

### Plugin ID 로 찾기 (함정 주의)

- 상단 본문 검색창(Search Vulnerabilities)은 이름/패밀리 텍스트만 검색한다. 여기 숫자(plugin ID)를 쳐도 "No records found" 가 뜬다
- plugin ID 로 찾으려면 `Filter ▼` → `Plugin ID` `is equal to` `<ID>` 로 조건 지정

### SMB 공유 확인

- `Microsoft Windows SMB Shares Enumeration` - 공유 목록 (ADMIN$, C$, IPC$ + 커스텀 공유)
- `Microsoft Windows SMB Shares Access` - 접근 가능 여부
- 기본 공유($ 붙은 것) 말고 커스텀 공유 이름이 보통 관심 대상

---

## 4. 리포트 내보내기 (UI 랙 우회)

취약점이 수백 개면 UI 가 무겁다. Export 로 빼서 보면 빠르다.

```
스캔 또는 취약점 화면 우측 상단 Export / Report
  - CSV      - Plugin ID, Name, Severity, Port 표. grep/엑셀로 바로 검색
  - .nessus  - Nessus 네이티브(다른 Nessus 로 import)
  - PDF/HTML - 보고용
```

```bash
# 내보낸 CSV 에서 답/항목 찾기
grep -i "26925"    export.csv     # plugin ID
grep -i "SMB Share" export.csv     # 공유
grep -i "VNC"       export.csv     # 서비스/포트
```

---

## 5. CLI 관리 (nessuscli)

```bash
# 서비스
sudo systemctl start nessusd
sudo systemctl status nessusd

# 사용자/업데이트
sudo /opt/nessus/sbin/nessuscli adduser
sudo /opt/nessus/sbin/nessuscli update --all
```

---

## 팁

- 인증 스캔이 핵심이다. 비인증은 표면만, 인증은 내부 패치/설정/공유까지 본다
- ALL ports 는 기본이 아니다. 전체 포트가 필요하면 Discovery 에서 꼭 바꾼다
- plugin ID 검색은 본문 검색창 말고 Filter(Plugin ID is equal to)
- 결과가 많아 UI 가 느리면 CSV Export 로 보는 게 빠르다
- severity 10.0 이 여러 개면 목록 맨 위가 최상위
- Nessus 는 "찾아주는" 도구다. 나온 취약점은 수동으로 검증/익스플로잇해야 실제 입증이 된다