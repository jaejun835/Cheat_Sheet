# 02_hashcat

# hashcat

---

## 개념

hashcat은 GPU 가속을 활용한 세계에서 가장 빠른 패스워드 크래킹 툴이다. 300개 이상의 해시 타입을 지원하며 딕셔너리, 마스크, 룰 기반 등 다양한 공격 방식을 제공한다. VM 환경에서는 GPU 패스스루가 제대로 안 되어 CPU로만 동작할 수 있어 `--force`가 필요한 경우가 있다.

---

## 기본 문법 / 옵션

```bash
hashcat -m <모드> -a <공격방식> <해시파일> <워드리스트/마스크>

# 자주 쓰는 옵션
-m              해시 모드 번호
-a              공격 방식 (0=딕셔너리, 1=조합, 3=마스크, 6=딕셔너리+마스크, 7=마스크+딕셔너리)
-r              룰 파일
-o              결과 저장 파일
--show          이미 크랙된 결과 확인
--username      해시 파일에 user:hash 형식일 때 유저명 무시
--force         경고 무시 (VM 환경에서 GPU 인식 안 될 때)
-w 3            워크로드 (1=Low, 2=Default, 3=High, 4=Nightmare)
-O              최적화 커널 (32자 이하 패스워드, 속도 향상)
--status        진행 상황 주기적 출력
--status-timer  상태 출력 주기 (초 단위)
--session       세션 이름 지정
--restore       중단된 세션 재개
-b              벤치마크 (속도 측정)

# 벤치마크
hashcat -b -m 1000    # NTLM 속도 확인
hashcat -b -m 3200    # bcrypt 속도 확인
hashcat -b            # 전체 모드 벤치마크
```

---

## 공격 방식 (-a)

```bash
# -a 0: 딕셔너리 공격 (가장 기본)
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt

# -a 0 + 룰: 딕셔너리 + 변형 규칙
hashcat -m 1000 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# -a 1: 딕셔너리 조합 (두 워드리스트의 모든 조합)
hashcat -m 0 hash.txt -a 1 list1.txt list2.txt

# -a 3: 마스크(브루트포스)
# 문자셋: ?l=소문자  ?u=대문자  ?d=숫자  ?s=특수문자  ?a=전체(?l?u?d?s)
hashcat -m 0 hash.txt -a 3 '?a?a?a?a?a?a?a?a'    # 8자리 전체
hashcat -m 0 hash.txt -a 3 'Pass?d?d?d?d'          # Pass + 숫자4자리
hashcat -m 0 hash.txt -a 3 '?u?l?l?l?d?d?d?s'      # 대+소3+숫자3+특수1

# 커스텀 문자셋 (-1 ~ -4)
hashcat -m 0 hash.txt -a 3 -1 ?l?d '?1?1?1?1?1?1'  # 소문자+숫자 6자리
hashcat -m 0 hash.txt -a 3 -1 ?u?l -2 ?d?s '?1?1?1?1?2?2'

# -a 6: 딕셔너리 + 마스크 (단어 뒤에 패턴 붙이기)
hashcat -m 0 hash.txt -a 6 rockyou.txt '?d?d?d?d'   # 단어+숫자4

# -a 7: 마스크 + 딕셔너리 (단어 앞에 패턴 붙이기)
hashcat -m 0 hash.txt -a 7 '?d?d?d?d' rockyou.txt   # 숫자4+단어
```

---

## 해시 모드 번호 전체 정리

```
# Windows / AD
1000    NTLM
5500    NetNTLMv1
5600    NetNTLMv2
1100    Domain Cached Credentials (DCC / MS-Cache v1) — XP/2003
2100    Domain Cached Credentials 2 (DCC2 / MS-Cache v2) — Vista+ (매우 느림)
13100   Kerberos TGS-REP RC4 (Kerberoast)
19600   Kerberos TGS-REP AES128
19700   Kerberos TGS-REP AES256
18200   Kerberos AS-REP RC4 (AS-REP Roast)
19800   Kerberos AS-REP AES128
19900   Kerberos AS-REP AES256

# 리눅스
500     MD5crypt $1$
1800    SHA512crypt $6$ (현대 리눅스 기본)
7400    SHA256crypt $5$
25600   yescrypt $y$ (최신 리눅스)

# 웹 / 애플리케이션
0       MD5
100     SHA1
1400    SHA256
1700    SHA512
900     MD4
400     phpass $P$ / $H$ (WordPress, phpBB)
3200    bcrypt $2*$
1600    Apache MD5 $apr1$ (.htpasswd)
10000   Django PBKDF2-SHA256
7900    Drupal7

# 데이터베이스
200     MySQL323
300     MySQL4.1/MySQL5+ (*로 시작)
131     MSSQL (2000)
132     MSSQL (2005)
1731    MSSQL (2012/2014)
12      PostgreSQL
3100    Oracle H: 구버전

# 문서 / 파일
10400   PDF 1.1–1.3 (Acrobat 2~4)
10500   PDF 1.4–1.6 RC4 128-bit (Acrobat 5~8)
10600   PDF 1.7 Level 3 (Acrobat 9)
10700   PDF 1.7 Level 8 AES-256 (Acrobat 10+)
9700    MS Office <= 2003 $0/$1 MD5+RC4
9800    MS Office <= 2003 $3/$4 SHA1+RC4
9400    MS Office 2007
9500    MS Office 2010
9600    MS Office 2013 / 2016 / 2019
25300   MS Office 2016 SheetProtection

# 아카이브
13600   WinZip AES
17200   PKZIP (압축)
17210   PKZIP (비압축)
17220   PKZIP (혼합)
11600   7-Zip
13000   RAR5
23800   RAR3-hp

# 비밀번호 관리자
13400   KeePass 1.x / 2.x
5200    Password Safe v3 (psafe3)
16900   Ansible Vault

# Cisco / 네트워크
500     Cisco Type 5 ($1$)
9200    Cisco Type 8 ($8$)
9300    Cisco Type 9 ($9$)
5700    Cisco IOS SHA256

# Wifi
22000   WPA-PBKDF2-PMKID+EAPOL (최신)
2500    WPA/WPA2 hccapx (구버전)

# macOS
7100    macOS v10.8+ (PBKDF2-SHA512)
```

---

## 룰 조합

룰은 워드리스트의 각 단어에 변형을 적용한다. 단어 끝에 숫자 추가, 대소문자 변경, 특수문자 추가 등이다.

```bash
# 기본 — 대부분 커버
hashcat -m <mode> hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# 좀 더 공격적
hashcat -m <mode> hash.txt rockyou.txt -r /usr/share/hashcat/rules/d3ad0ne.rule

# 룰 체이닝 (순서대로 적용)
hashcat -m <mode> hash.txt rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule \
  -r /usr/share/hashcat/rules/toggles1.rule

# OneRuleToRuleThemAll (강력, 느림)
hashcat -m <mode> hash.txt rockyou.txt \
  -r /usr/share/hashcat/rules/OneRuleToRuleThemAll.rule

# rockyou-30000
hashcat -m <mode> hash.txt rockyou.txt \
  -r /usr/share/hashcat/rules/rockyou-30000.rule

# 룰 파일 목록 확인
ls /usr/share/hashcat/rules/

# 결과 확인
hashcat -m <mode> hash.txt --show
```