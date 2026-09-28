# 03_office

# 파일 포맷 분석 — Office 문서 (Word / Excel / PowerPoint)

---

## 개념

Office 문서는 메타데이터(작성자, 수정자, 회사명 등)와 매크로 분석에 활용된다. 메타데이터에서 내부 유저명을 발견하면 계정 열거나 패스워드 스프레이에 활용할 수 있다. 매크로가 포함된 문서는 악성 매크로 분석이나 임베디드 페이로드 탐지에 쓴다.

파일 포맷 구분:
- 구버전 `.doc` `.xls` `.ppt` → OLE2 바이너리 형식 (매직바이트: `D0 CF 11 E0`)
- 신버전 `.docx` `.xlsx` `.pptx` → ZIP + XML 형식 (매직바이트: `50 4B 03 04`)

---

## 메타데이터 추출 (exiftool)

```bash
# 기본 메타데이터 확인
exiftool document.docx
exiftool spreadsheet.xlsx
exiftool presentation.pptx

# 유저명 수집 목적 — 작성자/수정자/회사명만 추출
# 발견된 이름을 AD 계정 스프레이에 활용 가능
exiftool -Author -Creator -LastModifiedBy -Company document.docx

# 여러 파일 한번에 처리
exiftool *.docx *.xlsx *.doc *.xls 2>/dev/null | grep -i "author\|creator\|modified by\|company\|last saved"

# JSON 형식으로 출력 (파싱에 유용)
exiftool -j document.docx

# 구버전 Office 파일 (.doc, .xls, .ppt — OLE2 형식)
exiftool document.doc
```

---

## 텍스트 내용 추출 및 탐색

```bash
# docx/xlsx/pptx는 ZIP 구조 — 직접 압축 해제해서 XML 읽기 가능
unzip document.docx -d docx_contents

# Word 본문 텍스트 (XML 태그 제거)
cat docx_contents/word/document.xml | sed 's/<[^>]*>//g' | tr -s ' \n' '\n' | grep -v "^$"

# Excel 시트 데이터
cat docx_contents/xl/sharedStrings.xml | sed 's/<[^>]*>//g' | grep -v "^$"

# 자격증명 키워드 탐색
cat docx_contents/word/document.xml | grep -i "pass\|pwd\|secret\|token\|credential"

# strings로 바이너리에서 평문 추출 (구버전 doc 포함)
strings document.doc | grep -i "pass\|pwd\|http\|user\|admin\|credential"
strings document.docx | grep -i "pass\|pwd\|user"

# grep으로 여러 파일 한번에 탐색
grep -r -i "password\|passwd\|credential" docx_contents/
```

---

## 매크로 분석 (oletools)

```bash
# 설치
pip install oletools --break-system-packages

# 매크로 존재 여부 확인 및 내용 추출
# VBA 매크로, 자동 실행 매크로, 의심스러운 키워드 탐지
olevba document.doc
olevba document.docm
olevba spreadsheet.xlsm

# 매크로 내용만 출력 (분석 없이)
olevba --reveal document.doc

# 자동 실행 함수만 확인 (AutoOpen, AutoExec, Document_Open 등)
olevba --show-attrs document.doc

# 악성 여부 빠른 판단 (SUSPICIOUS, HIGH_RISK 키워드 탐지)
mraptor document.doc

# OLE 오브젝트 내 임베디드 파일 추출 (아이콘으로 숨겨진 실행 파일 등)
# 추출된 파일은 현재 디렉토리에 저장됨
oleobj document.doc

# RTF 파일 내 OLE 오브젝트 확인
rtfobj document.rtf
```

---

## 암호화 Office 문서

암호화된 Office 문서 크래킹은 `07_password_cracking` 참조.

```bash
# 암호화 여부 확인
file document.docx
# "Composite Document File" 이면 OLE2 → 구버전 또는 암호화된 신버전

# office2john으로 해시 추출
office2john document.docx > office.hash

# 버전 확인 ($office$*연도* 부분)
cat office.hash | grep -o '\$office\$\*[0-9]*'
```