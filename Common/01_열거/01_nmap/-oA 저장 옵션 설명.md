# -oA 저장 옵션 설명

### .nmap

```
PORT      STATE SERVICE
21/tcp    open  ftp
53/tcp    open  domain
88/tcp    open  kerberos-sec
```

터미널에서 보이는 출력을 그대로 저장하는 형식이다

 헤더, 주석, 요약까지 전부 포함되어 있어서 스캔 결과를 눈으로 확인하거나 나중에 다시 참고할 때 제일 읽기 편하다 

### **.gnmap**

```
Host: 10.129.228.6 ()  Ports: 21/open/tcp//ftp///, 53/open/tcp//domain///
```

호스트 하나당 한 줄에 모든 포트 정보를 (포트번호/상태/프로토콜/)서비스 형식으로 압축해서 저장한 grep이나 awk로 열린 포트 번호만 빠르게 추출할 수 있어서 다음 스캔 명령어 만들 때 바로 활용하기 좋다 

### **.xml**

```xml
<port protocol="tcp" portid="21">
  <state state="open"/>
  <service name="ftp"/>
</port>
```

XML 구조로 저장되기 때문에 사람이 직접 읽기엔 불편하지만 다른 도구와 연동할 때 유용하다Metasploit에서 db_import ports.xml 하면 스캔 결과가 자동으로 DB에 들어가고, Python이나 다른 자동화 스크립트에서 파싱할 때도 이 파일을 주로 사용한다