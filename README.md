# Cheat Sheet

해킹 실습과 보안 학습 과정에서 정리한 개인 치트시트입니다. 열거, 서비스별 공격, 쉘 획득, 파일 전송, 권한 상승부터 Active Directory 공격 흐름까지 주제별로 정리합니다.

명령어를 빠르게 찾아보는 참고 자료이자, 각 기법의 사용 조건과 동작 원리를 복습하기 위한 학습 노트입니다.

## Browse by Area

- [Playbooks](playbooks/README.md)
- [Web](web/README.md)
- [Linux](linux/README.md)
- [Windows](windows/README.md)
- [Active Directory](active-directory/README.md)
- [Common](common/README.md)

## Repository Structure

| 분야 | 주요 내용 |
|---|---|
| **Playbooks** | CTF, Bug Bounty, Pentest 상황별 탐색 목차 |
| **Common** | 여러 환경에 공통으로 쓰는 자료와 분야를 가로지르는 목차 |
| **Web** | 웹 열거, HTTP 취약점, 제품별 자료 |
| **Linux** | Linux 열거와 권한 상승 |
| **Windows** | Windows 권한 상승과 관련 자료 연결 |
| **Active Directory** | 외부 접근, 내부 접근, DC 관련 자료 |

```text
Cheat_Sheet/
├── README.md
├── playbooks/
│   ├── README.md
│   ├── ctf.md
│   ├── bug-bounty.md
│   └── pentest.md
├── common/
│   ├── enumeration/
│   ├── services/
│   ├── shells/
│   ├── file-transfer/
│   ├── permissions/
│   ├── privilege-escalation/
│   ├── password-cracking/
│   ├── file-formats/
│   ├── tools/
│   └── pivoting-and-tunneling.md
├── web/
│   ├── enumeration/
│   └── http/
├── linux/
│   ├── enumeration/
│   └── privilege-escalation/
├── windows/
│   └── privilege-escalation/
└── active-directory/
    ├── external-access/
    ├── internal-access/
    └── domain-controller/
```

## Common

[Common](common/README.md)

| 주제 | 정리 내용 |
|---|---|
| [Enumeration](common/enumeration/README.md) | Nmap, enum4linux, 디렉터리 탐색, DNS 열거, JavaScript 분석 등 |
| [Services](common/services/README.md) | FTP, SSH, SMB, HTTP/HTTPS 등 서비스별 점검·공격 기법 |
| [Shells](common/shells/README.md) | 리버스 쉘과 쉘 업그레이드 |
| [File Transfer](common/file-transfer/README.md) | Linux·Windows 환경 간 파일 전송 방법 |
| [Permissions](common/permissions/README.md) | 권한 정보를 읽고 해석하는 방법 |
| [Privilege Escalation](common/privilege-escalation/README.md) | Linux·Windows 권한 상승 기법 |
| [Password Cracking](common/password-cracking/README.md) | 해시 식별, Hashcat, John the Ripper, 해시 유형별 크래킹 |
| [File Formats](common/file-formats/README.md) | MDB, PST, Office, PDF, EML, 압축파일 등의 분석 |
| [Pivoting and Tunneling](common/pivoting-and-tunneling.md) | 내부 네트워크 접근을 위한 피버팅과 터널링 |
| [Tool Troubleshooting](common/tools/tool-troubleshooting.md) | 도구 사용 중 발생하는 오류와 해결 방법 |
| [Metasploit](common/tools/metasploit.md) | Metasploit 사용법과 관련 명령어 |

## Active Directory

[Active Directory](active-directory/README.md)

| 단계 | 정리 내용 |
|---|---|
| [External Access](active-directory/external-access/README.md) | 사용자 열거, AS-REP Roasting, 패스워드 스프레이, Kerberoasting, SMB Relay |
| [Internal Access](active-directory/internal-access/README.md) | BloodHound, Pass-the-Hash, Pass-the-Ticket, 권한 관계별 공격, 위임 설정 악용 등 |
| [Domain Controller](active-directory/domain-controller/README.md) | Golden Ticket, Silver Ticket, 지속성 관련 기법 |

단계 구분은 노트를 탐색하기 위한 분류입니다. 실제 사용 가능한 기법과 진행 순서는 확보한 계정, 권한, 서비스 설정에 따라 달라집니다.

## How to Use

- **특정 도구나 명령어가 필요할 때:** 위 목차에서 해당 주제로 이동합니다.
- **전체 흐름을 복습할 때:** 공통 치트시트는 열거부터, AD 노트는 외부 침투부터 순서대로 살펴봅니다.
- **하위 주제를 찾을 때:** 각 폴더의 `README.md`에서 관련 문서를 확인합니다.
- **명령어를 실행할 때:** 예시에 나온 IP, 도메인, 계정, 경로 등을 자신의 실습 환경에 맞게 변경하고 필요한 조건을 먼저 확인합니다.

## About These Notes

노션에서 작성한 학습 노트를 GitHub에서 읽기 편하도록 재구성했습니다. 학습과 실습 과정에서 내용을 보완하며, 도구 버전이나 대상 환경에 따라 명령어와 결과가 달라질 수 있습니다.

이 자료는 본인 소유 또는 명시적으로 허가받은 환경에서의 보안 학습과 테스트를 목적으로 합니다.

## License

[LICENSE](LICENSE)
