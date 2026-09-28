# 04_pdf

# 파일 포맷 분석 — PDF

---

## 개념

PDF 파일은 메타데이터에서 작성자, 소프트웨어, 회사명 등을 확인하거나 첨부파일을 추출하는 데 활용한다. 암호화된 PDF 크래킹은 `07_password_cracking` 참조.

---

## 메타데이터 추출

```bash
# exiftool로 기본 메타데이터 확인
exiftool document.pdf

# 작성자, 소프트웨어, 회사명만 추출 (유저명 수집)
exiftool document.pdf | grep -i "author\|creator\|producer\|company\|subject\|keywords"

# 여러 PDF 한번에
exiftool *.pdf | grep -i "author\|creator\|company"

# pdfinfo로 메타데이터 확인 (poppler-utils 포함)
sudo apt install poppler-utils
pdfinfo document.pdf
```

---

## 텍스트 추출 및 탐색

```bash
# PDF에서 텍스트 추출
pdftotext document.pdf -          # stdout 출력
pdftotext document.pdf output.txt  # 파일 저장

# 자격증명 키워드 탐색
pdftotext document.pdf - | grep -i "pass\|pwd\|user\|login\|secret\|token\|api"

# 내부 IP, URL 추출
pdftotext document.pdf - | grep -oE "192\.168\.[0-9]+\.[0-9]+"
pdftotext document.pdf - | grep -oE "https?://[a-zA-Z0-9./_-]+"

# 페이지별 텍스트 추출
pdftotext -f 1 -l 1 document.pdf -   # 1페이지만
pdftotext -f 2 -l 5 document.pdf -   # 2~5페이지

# strings로 평문 직접 탐색
strings document.pdf | grep -i "pass\|pwd\|user\|admin"
```

---

## 첨부파일 / 임베디드 파일 추출

```bash
# PDF 내 첨부파일 목록 확인
pdfdetach -list document.pdf

# 모든 첨부파일 현재 디렉토리에 저장
pdfdetach -saveall document.pdf

# binwalk로 임베디드 파일 탐지 (이미지, 문서 등)
binwalk document.pdf

# 임베디드 파일 추출 (_document.pdf.extracted/ 에 저장)
binwalk -e document.pdf

# 추출된 파일 확인
ls _document.pdf.extracted/ 2>/dev/null || ls _*.extracted/ 2>/dev/null
```

---

## 암호화 PDF

```bash
# 암호화 여부 확인
pdfinfo document.pdf | grep -i "encrypt"

# pdf2john으로 해시 추출 후 크래킹
pdf2john document.pdf > pdf.hash
cat pdf.hash
# 크래킹은 07_password_cracking 참조
```