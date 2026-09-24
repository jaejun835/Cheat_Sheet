# 방화벽/IDS 우회

일반 스캔에서 포트가 filtered로 나오거나 방화벽 규칙,IDS 탐지를 우회해야 할 때 선택적으로 사용한다

외부에서는 정확한 방화벽 규칙을 미리 확인하기 어렵기 때문에 문제의 단서와 스캔 증상에 따라 가능성이 높은 기법부터 하나씩 비교한다

즉 여러 기법을 처음부터 한꺼번에 섞으면 어떤 조건이 효과가 있었는지 할 수 없다

```bash
# ------------------------------------------------------------
# 1. 신뢰된 출발지 포트 이용
#
# 사용 상황:
# 방화벽이 DNS, HTTP, FTP 같은 특정 서비스의 응답 트래픽을 신뢰하여
# 해당 출발지 포트에서 온 패킷을 허용한다고 의심될 때 사용한다.
#
# 판단 방법:
# 일반 스캔에서 filtered였던 포트가 특정 source port를 사용했을 때
# open으로 바뀌면 해당 출발지 포트를 허용하는 규칙을 의심할 수 있다.
# ------------------------------------------------------------

# DNS 응답 트래픽으로 위장
sudo nmap -Pn -sS -p- \
  --source-port 53 \
  <IP> -oA source53

# 짧은 표기: --source-port 53과 동일
sudo nmap -Pn -sS -p- \
  -g 53 \
  <IP> -oA source53_short

# HTTP 출발지 포트를 신뢰하는 규칙이 의심될 때
sudo nmap -Pn -sS -p- \
  --source-port 80 \
  <IP> -oA source80

# FTP active mode 관련 source port 20 허용 규칙이 의심될 때
sudo nmap -Pn -sS -p- \
  --source-port 20 \
  <IP> -oA source20

# UDP 서비스가 출발지 포트 기준으로 필터링되는지 확인
sudo nmap -Pn -sU -p- \
  --source-port 53 \
  <IP> -oA udp_source53

# ------------------------------------------------------------
# 2. IP 패킷 단편화
#
# 사용 상황:
# 오래된 방화벽이나 IDS가 조각난 IP 패킷을 정상적으로 재조립하지 못하거나,
# TCP/UDP 헤더가 여러 조각으로 나뉘면 탐지 규칙을 적용하지 못한다고 의심될 때 사용한다.
#
# 중요:
# 단편화는 주로 raw 패킷을 사용하는 포트 발견 단계에 적용된다.
# -sV와 NSE의 애플리케이션 통신에는 그대로 적용되지 않을 수 있다.
# ------------------------------------------------------------

# 8바이트 단위로 데이터 단편화
sudo nmap -Pn -sS -p- \
  -f \
  <IP> -oA fragment8

# 16바이트 단위로 데이터 단편화
sudo nmap -Pn -sS -p- \
  -f -f \
  <IP> -oA fragment16

# UDP 포트 필터링에도 단편화가 영향을 주는지 확인
sudo nmap -Pn -sU -p- \
  -f \
  <IP> -oA udp_fragment

# 운영체제가 단편을 다시 조립해 전송하는 문제가 있을 때 Ethernet 전송 강제
sudo nmap -Pn -sS -p- \
  -f --send-eth \
  <IP> -oA fragment_sendeth

# ------------------------------------------------------------
# 3. MTU 직접 지정
#
# 사용 상황:
# -f의 고정 단편 크기 대신 직접 단편 크기를 조정하여
# 특정 크기의 패킷만 방화벽이나 IDS를 통과하는지 비교할 때 사용한다.
#
# 중요:
# MTU는 8의 배수로 지정하며 -f와 동시에 사용하지 않는다.
# ------------------------------------------------------------

# MTU 24로 단편화
sudo nmap -Pn -sS -p- \
  --mtu 24 \
  <IP> -oA mtu24

# 다른 단편 크기와 비교
sudo nmap -Pn -sS -p- \
  --mtu 32 \
  <IP> -oA mtu32

# UDP에서도 동일한 단편 크기로 확인
sudo nmap -Pn -sU -p- \
  --mtu 24 \
  <IP> -oA udp_mtu24

# ------------------------------------------------------------
# 4. Decoy 스캔
#
# 사용 상황:
# IDS 로그에 실제 스캐너 IP와 여러 가짜 출발지 IP를 함께 남겨
# 실제 출발지를 식별하기 어렵게 만들 때 사용한다.
#
# 중요:
# Decoy는 포트를 open으로 만드는 직접적인 방화벽 우회 기법이 아니다.
# 스캔 결과만으로 성공 여부를 확인하기 어렵고 IDS 로그 접근이 있어야 정확히 검증할 수 있다.
# ------------------------------------------------------------

# 무작위 decoy 10개 사용
sudo nmap -Pn -sS -p- \
  -D RND:10 \
  <IP> -oA decoy_random

# decoy IP와 실제 내 IP의 위치를 직접 지정
# ME가 실제 내 IP를 의미한다.
sudo nmap -Pn -sS -p- \
  -D <DECOY1>,ME,<DECOY2> \
  <IP> -oA decoy_manual

# 무작위 decoy 사이에 실제 내 IP 삽입
sudo nmap -Pn -sS -p- \
  -D RND:5,ME \
  <IP> -oA decoy_mixed

# ------------------------------------------------------------
# 5. 느린 스캔과 요청 간격 조절
#
# 사용 상황:
# IDS/IPS가 일정 시간 동안 발생한 패킷 수, 접근한 포트 수,
# 연결 시도 횟수를 기준으로 스캐너를 차단한다고 의심될 때 사용한다.
#
# 판단 방법:
# 빠른 스캔은 중간부터 응답이 사라지거나 filtered가 증가하지만,
# 느린 스캔에서는 응답이 유지되면 rate-based 탐지나 일시 차단을 의심할 수 있다.
# ------------------------------------------------------------

# 각 probe 사이에 최소 1초 대기
sudo nmap -Pn -sS -p- \
  --scan-delay 1s \
  <IP> -oA delay1s

# 특정 포트만 더 느리게 확인
sudo nmap -Pn -sS \
  --scan-delay 2s \
  -p<PORTS> \
  <IP> -oA delay_selected

# Sneaky 타이밍 템플릿
sudo nmap -Pn -sS -p- \
  -T1 \
  <IP> -oA timing_T1

# Paranoid 타이밍 템플릿
# 매우 느리므로 좁은 포트 범위에서 사용하는 편이 현실적이다.
sudo nmap -Pn -sS \
  -T0 \
  -p<PORTS> \
  <IP> -oA timing_T0

# Nmap이 자동으로 적용하는 probe 간 지연의 최대값 제한
# 최소 지연을 강제하는 옵션은 아니다.
sudo nmap -Pn -sS -p- \
  --max-scan-delay 5s \
  <IP> -oA max_delay

# ------------------------------------------------------------
# 6. 패킷 크기와 데이터 변경
#
# 사용 상황:
# IDS가 Nmap의 기본 패킷 크기나 고정된 payload 패턴을
# 시그니처로 사용해 탐지한다고 의심될 때 사용한다.
#
# 판단 방법:
# 기본 패킷은 차단되지만 패킷 길이나 payload를 변경했을 때
# 응답이 나타나면 크기 또는 시그니처 기반 탐지를 의심할 수 있다.
# ------------------------------------------------------------

# 각 패킷에 랜덤 데이터 25바이트 추가
sudo nmap -Pn -sS -p- \
  --data-length 25 \
  <IP> -oA data25

# 다른 패킷 크기와 비교
sudo nmap -Pn -sS -p- \
  --data-length 50 \
  <IP> -oA data50

# UDP 패킷에도 랜덤 데이터 추가
sudo nmap -Pn -sU -p- \
  --data-length 25 \
  <IP> -oA udp_data25

# 지정한 문자열을 payload에 추가
sudo nmap -Pn -sS -p- \
  --data-string "normal-traffic" \
  <IP> -oA data_string

# 지정한 16진수 데이터를 payload에 추가
sudo nmap -Pn -sS -p- \
  --data 0xdeadbeef \
  <IP> -oA data_hex

# ------------------------------------------------------------
# 7. 잘못된 체크섬 전송
#
# 사용 상황:
# 대상 운영체제가 아니라 중간의 방화벽, IDS, 프록시 또는 로드밸런서가
# 대신 응답하는지 확인할 때 사용하는 진단 기법이다.
#
# 중요:
# 정상 운영체제는 잘못된 체크섬의 패킷을 폐기해야 한다.
# 따라서 실제 열린 포트를 찾기 위한 일반적인 우회 스캔으로 사용하지 않는다.
# ------------------------------------------------------------

# TCP 패킷에 잘못된 체크섬 적용
sudo nmap -Pn -sS -p- \
  --badsum \
  <IP> -oA badsum_tcp

# UDP 패킷에 잘못된 체크섬 적용
sudo nmap -Pn -sU -p- \
  --badsum \
  <IP> -oA badsum_udp

# ------------------------------------------------------------
# 8. TCP 플래그 스캔
#
# 사용 상황:
# 방화벽이 새로운 연결을 만드는 SYN 패킷은 차단하지만,
# ACK, FIN, NULL, Xmas 같은 다른 TCP 플래그에는 느슨한 규칙을 적용한다고 의심될 때 사용한다.
#
# 중요:
# ACK 스캔은 포트의 open/closed 판별이 아니라 filtered/unfiltered 판별용이다.
# FIN/NULL/Xmas의 open|filtered는 실제 open과 패킷 드롭을 구별하지 못한다.
# Windows 계열은 RFC 방식과 다르게 모든 포트에 RST를 보낼 수 있다.
# ------------------------------------------------------------

# ACK 패킷이 방화벽을 통과하는지 확인
sudo nmap -Pn -sA -p- \
  <IP> -oA ack_scan

# FIN 플래그만 전송
sudo nmap -Pn -sF -p- \
  <IP> -oA fin_scan

# TCP 플래그를 설정하지 않은 패킷 전송
sudo nmap -Pn -sN -p- \
  <IP> -oA null_scan

# FIN, PSH, URG 플래그를 함께 전송
sudo nmap -Pn -sX -p- \
  <IP> -oA xmas_scan

# 특정 포트의 필터링 여부만 빠르게 확인
sudo nmap -Pn -sA \
  -p<PORTS> \
  <IP> -oA ack_selected

# ------------------------------------------------------------
# 9. Idle / Zombie 스캔
#
# 사용 상황:
# 대상에 실제 내 IP가 아닌 zombie 호스트의 IP가 기록되도록 하거나,
# zombie와 대상 사이에만 허용된 방화벽 신뢰 관계를 확인할 때 사용한다.
#
# 중요:
# TCP 전용이며 예측 가능한 IP ID를 사용하는 거의 유휴 상태의 zombie가 필요하다.
# 적절한 zombie를 찾기 어려우므로 일반적인 첫 번째 우회 방법은 아니다.
# ------------------------------------------------------------

# zombie 후보의 IP ID 동작 확인
sudo nmap -Pn -O -v \
  <ZOMBIE_IP>

# zombie를 통해 특정 포트 확인
sudo nmap -Pn -sI <ZOMBIE_IP> \
  -p<PORTS> \
  <TARGET_IP> -oA idle_selected

# 조건이 맞을 때 전체 TCP 포트 확인
sudo nmap -Pn -sI <ZOMBIE_IP> \
  -p- \
  <TARGET_IP> -oA idle_scan

# ------------------------------------------------------------
# 10. 특정 인터페이스와 출발지 주소 강제
#
# 사용 상황:
# VPN, 다중 NIC 또는 피벗 환경에서 Nmap이 잘못된 인터페이스나
# 출발지 주소를 선택하여 패킷이 의도한 경로로 가지 않을 때 사용한다.
#
# 중요:
# 이것은 IDS 우회 자체보다는 잘못된 라우팅이나 인터페이스 선택 때문에
# 스캔 결과가 누락되는 것을 방지하는 옵션이다.
# ------------------------------------------------------------

# HTB VPN 인터페이스를 명시적으로 사용
sudo nmap -Pn -sS -p- \
  -e tun0 \
  <IP> -oA via_tun0

# 출발지 IP와 인터페이스를 함께 지정
# 지정한 SOURCE_IP가 실제로 해당 인터페이스에서 사용 가능해야 한다.
sudo nmap -Pn -sS -p- \
  -S <SOURCE_IP> \
  -e tun0 \
  <IP> -oA forced_source

# ------------------------------------------------------------
# 11. 우회 방식에서 찾은 포트의 서비스 버전 확인
#
# 사용 상황:
# source port 우회로 새 포트를 찾았지만 일반 -sV 또는 -sC에서는
# tcpwrapped, TIMEOUT, filtered 등이 나오는 경우 사용한다.
#
# 중요:
# Nmap의 --source-port는 SYN/UDP 포트 스캔에는 적용되지만,
# -sV 버전 탐지와 NSE의 애플리케이션 통신에는 적용되지 않는다.
# 따라서 연결 내내 source port를 유지해야 하면 Ncat/Nc로 직접 확인해야 한다.
# ------------------------------------------------------------

# 우회 조건을 유지하여 포트가 실제로 열려 있는지 재확인
sudo nmap -Pn -sS \
  --source-port 53 \
  -p<NEW_PORTS> \
  <IP> -oA hidden_port_check

# 일반 버전 탐지 시도
# 방화벽이 최초 SYN에만 source port 규칙을 적용하면 성공할 수 있다.
sudo nmap -Pn -sV \
  --version-all \
  -p<NEW_PORTS> \
  <IP> -oA hidden_version_normal

# 일반 -sV가 tcpwrapped 또는 TIMEOUT이면
# 실제 TCP 연결의 로컬 출발지 포트를 53으로 고정해 직접 배너 확인
sudo ncat -nv \
  --source-port 53 \
  <IP> <PORT>

# 서버가 먼저 배너를 보내지 않으면 빈 줄 전송
printf '\r\n' \
  | sudo ncat \
      --source-port 53 \
      <IP> <PORT>

# HTTP 또는 HTTP 계열 서비스인지 확인
printf 'GET / HTTP/1.0\r\nHost: <IP>\r\n\r\n' \
  | sudo ncat \
      --source-port 53 \
      <IP> <PORT>

# 일반 Netcat 사용
# -p 53은 로컬 출발지 포트를 53으로 지정한다.
sudo nc -nv \
  -p 53 \
  <IP> <PORT>

# UDP 서비스에 source port 53으로 직접 연결
sudo ncat -nvu \
  --source-port 53 \
  <IP> <PORT>

# 수신한 배너를 화면에 출력하면서 파일로 저장
sudo ncat -nv \
  --source-port 53 \
  <IP> <PORT> \
  | tee hidden_banner.txt
```

---