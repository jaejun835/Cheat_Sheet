# 12_CanRDP

Remote Desktop 접근 권한이다.

```bash
xfreerdp /v:<대상IP> /u:<공격자계정> /p:'<패스워드>' /dynamic-resolution  # ← 수정

# PtH RDP (Restricted Admin Mode 활성화 필요)
xfreerdp /v:<대상IP> /u:<공격자계정> /pth:<NTLM> /dynamic-resolution
```