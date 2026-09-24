# 02_enum4linux

# enum4linux

SMB/Samba에서 정보를 수집하는 Perl 기반 wrapper. 내부적으로 `smbclient`, `rpcclient`, `net`, `nmblookup`을 호출. 사용자·그룹·공유·OS·패스워드 정책·RID cycling 등을 한번에 열거.

---

## 옵션

| 옵션 | 설명 |
| --- | --- |
| `-a` | 전체 열거 (종합 스캔, 가장 자주 사용) |
| `-U` | 사용자 목록 |
| `-G` | 그룹 및 멤버 |
| `-S` | 공유 폴더 |
| `-P` | 패스워드 정책 |
| `-o` | OS 정보 |
| `-r` | RID cycling |
| `-u` / `-p` | 인증 사용자/패스워드 |
| `-d` | 상세 출력 |

---

## 명령어

```bash
# 표준 종합 스캔
enum4linux -a <IP> 2>&1 | tee enum4linux.txt

# 더 자세히
enum4linux -a -M -d <IP> 2>&1 | tee enum4linux.txt

# 인증 세션으로 열거
enum4linux -u 'username' -p 'password' -a <IP>

# RID cycling (null session 가능 시)
enum4linux -r <IP> | grep "Local User"

# 최신 Python 버전 (더 안정적, 권장)
enum4linux-ng -A <IP>
enum4linux-ng -A <IP> -oY results.yaml
enum4linux-ng -A <IP> -oA results      # JSON+YAML 동시 저장
```

---

## 출력 해석

| 항목 | 의미 및 다음 액션 |
| --- | --- |
| `Got domain/workgroup name` | AD 환경 가능성 → BloodHound 준비 |
| `[+] Sharing` 섹션 | 공유 폴더 → smbclient로 접근 |
| `Users via RID cycling` | 실제 계정 (RID 1000+) → 패스워드 스프레이 |
| `Password Policy` | 락아웃 임계값 → 스프레이 간격 결정 |
| `null session` 성공 | 익명 접근 가능 → 추가 열거 |

---

## smbclient 연계

```bash
# 공유 목록 (null session)
smbclient -L //<IP> -N

# 공유 접근
smbclient //<IP>/share -N
smbclient //<IP>/share -U 'user%password'

# 파일 다운로드
smb: \> get secret.txt
smb: \> mget *.txt
smb: \> recurse ON
smb: \> mget *
```

---

## netexec 대안 (더 빠름)

```bash
# 기본 정보
nxc smb <IP>

# 사용자 열거
nxc smb <IP> -u '' -p '' --users
nxc smb <IP> -u 'guest' -p '' --users

# 공유 열거
nxc smb <IP> -u '' -p '' --shares

# 패스워드 정책
nxc smb <IP> --pass-pol
nxc smb <IP> -u '' -p '' --pass-pol

# RID brute
nxc smb <IP> -u '' -p '' --rid-brute
nxc smb <IP> -u 'guest' -p '' --rid-brute 10000
```

---

## nmap NSE 보완

```bash
nmap -p 445 --script smb-enum-users,smb-enum-shares,smb-os-discovery,smb-security-mode <IP>
nmap -p 139,445 --script smb-vuln* <IP>    # 취약점 스캔
```

---

## 팁

- hang 걸리면 `enum4linux-ng` 로 대체
- null session 불가 시 → guest 계정 or 획득한 자격증명으로 재시도
- RID cycling으로 뽑은 사용자 목록 → 패스워드 스프레이 or AS-REP Roasting에 활용
- 445 포트가 열려있으면 SSH보다 enum4linux를 먼저 실행 (정보가 훨씬 많음)