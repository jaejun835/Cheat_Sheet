# 02_pst

# 파일 포맷 분석 — pst (Outlook Personal Folders)

---

## 개념

pst는 Microsoft Outlook의 이메일, 연락처, 일정 등을 저장하는 파일이다. 침투 테스트에서 발견 시 이메일 내용에 자격증명, 내부 정보, 첨부파일 등이 포함되어 있는 경우가 많다. 칼리에서 readpst(libpst)로 mbox 형식으로 변환한 뒤 분석한다.

Windows에서의 기본 저장 경로:
- Outlook 2010 이하: `%APPDATA%\Microsoft\Outlook\`
- Outlook 2013 이상: `%LOCALAPPDATA%\Microsoft\Outlook\`

---

## 기본 분석

```bash
# 설치
sudo apt install libpst-utils

# 파일 타입 확인
file archive.pst
# Microsoft Outlook email folder (>=2003)

# 파일 크기 확인 (큰 파일은 변환에 시간이 걸림)
ls -lh archive.pst

# 이메일 목록 빠르게 훑기 (변환 없이 발신자/제목/날짜 확인)
# 전체 변환 전에 어떤 내용이 있는지 파악할 때 유용
lspst archive.pst

# CC, BCC까지 포함해서 출력
lspst -l archive.pst

# 결과에서 흥미로운 제목 필터
lspst archive.pst | grep -i "pass\|cred\|access\|vpn\|ssh\|login\|account"
```

---

## 변환 및 내용 추출

```bash
# 기본 변환 — mbox 형식으로 변환 (같은 디렉토리에 생성)
# .mbox 파일명은 PST 내 폴더명 기반으로 자동 생성됨
readpst archive.pst
ls -la *.mbox

# 출력 디렉토리 지정
mkdir pst_output
readpst -o pst_output archive.pst

# 이메일을 개별 파일로 분리
# -S: 각 이메일이 별도 파일, PST 폴더 구조 그대로 유지
readpst -S -o pst_output archive.pst

# 이메일 + 첨부파일 분리 (-M: MH 형식)
# 각 이메일이 첨부파일과 함께 별도 파일로 저장됨
readpst -M -o pst_output archive.pst

# 삭제된 항목 포함 (삭제된 이메일에도 유용한 정보 있을 수 있음)
readpst -D -o pst_output archive.pst

# 이메일 타입만 추출 (연락처/일정 제외, 빠름)
readpst -t e -o pst_output archive.pst

# 재귀적으로 하위 폴더까지 처리
readpst -r -o pst_output archive.pst
```

---

## 자격증명 탐색

변환이 완료되면 mbox 파일에서 자격증명을 체계적으로 탐색한다.

```bash
# 변환된 mbox 파일 목록 확인
ls -lh pst_output/ 2>/dev/null || ls -lh *.mbox

# 전체 이메일 내용에서 자격증명 관련 키워드 검색
grep -r -i "password\|passwd\|pwd\|pass:" pst_output/
grep -r -i "username\|user name\|login\|credential" pst_output/
grep -r -i "secret\|token\|api.key\|ssh\|vpn" pst_output/

# mbox 파일 직접 검색
cat *.mbox | grep -i "password\|passwd\|credential"

# 이메일 헤더 정보 (발신자, 수신자, 날짜)
grep -r "^From:\|^To:\|^Subject:\|^Date:" pst_output/ | head -50

# 내부 IP, URL 추출 (내부 시스템 정보 파악)
grep -r -o "192\.168\.[0-9]\+\.[0-9]\+" pst_output/
grep -r -o "10\.[0-9]\+\.[0-9]\+\.[0-9]\+" pst_output/
grep -r -E "https?://[a-zA-Z0-9./_-]+" pst_output/ | head -30

# 첨부파일 목록 확인
find pst_output/ -type f ! -name "*.mbox" ! -name "*.txt" 2>/dev/null
```

---

## PST → 크레덴셜 흐름

```
PST 파일 발견
  → lspst로 이메일 목록 빠르게 훑기
  → readpst로 mbox 변환
  → grep으로 자격증명 키워드 탐색
  → 발견한 자격증명으로 다른 서비스 로그인 시도
    (FTP / Telnet / SSH / SMB / 웹)
  → 첨부파일이 있으면 별도 추출 후 분석
    (Office 문서 → 03_office.md 참조)
    (ZIP/RAR → 06_아카이브.md 참조)
```