# 04_파일전송

[Linux_to_Linux](Linux_to_Linux.md)

[Linux_to_Windows](Linux_to_Windows.md)

[Windows_to_Linux](Windows_to_Linux.md)

## 환경 메모 (실전 함정)

- 리스너 IP 는 tun0 가 없을 수 있다. WSL 등에선 `ip -4 addr` 로 10.x 또는 실제 VPN 인터페이스를 직접 확인
- WSL 환경은 인바운드가 막힌다. 타겟이 칼리로 접속하는 방식(HTTP/SMB/certutil/FTP)이 전부 실패할 수 있다
  - 확인: 타겟에서 `curl http://<칼리IP>/` → "Unable to connect" 면 인바운드 차단
  - 전환: 아웃바운드 방식(SCP) 또는 RDP 드라이브 공유(`/drive:` → `\\tsclient\`)로
- 리눅스 타겟은 SSH(22) 열려 있으면 SCP 가 가장 깔끔하다 (아웃바운드라 인바운드 문제 무관)
- WSL↔윈도우 파일은 `/mnt/c/...` 로 바로 접근 (윈도우 다운로드 폴더를 칼리에서 cp)
