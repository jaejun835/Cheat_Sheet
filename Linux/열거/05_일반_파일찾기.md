## 이름 / 경로로 찾기

`-name` 은 파일 이름만 비교하고, `-path` 는 전체 경로를 비교한다. 찾는 패턴에 `/` 가 들어가면 `-name` 으로는 절대 안 걸린다.

```
# 파일 이름으로 찾기
find / -name "<file>" -type f 2>/dev/null

# 대소문자 무시
find / -iname "<file>" -type f 2>/dev/null

# 경로 일부로 찾기 (앞에 * 필수)
find / -path "*<dir>/<file>" 2>/dev/null

# 검색 범위 좁히기 (워드리스트는 보통 /usr/share 아래)
find /usr/share -path "*Web-Content/common.txt" 2>/dev/null

# 깊이 제한
find / -maxdepth 3 -name "<file>" 2>/dev/null

# 다른 파일시스템으로 안 넘어가기 (WSL의 /mnt/c 등 제외)
find / -xdev -name "<file>" 2>/dev/null

# 특정 경로 제외
find / -path /mnt -prune -o -name "<file>" -print 2>/dev/null

# 인덱스 DB로 즉시 찾기 (처음엔 sudo updatedb 필요)
locate <file>
```

주의사항:

- `-name` 에 `/` 가 들어가면 결과 0개 → `-path` 사용
- `2>/dev/null` 은 find 자체 경고까지 숨김 → 결과가 이상하게 비면 `2>/dev/null` 빼고 다시 실행해서 경고 확인
- WSL에서 `/` 부터 찾으면 `/mnt/c` (Windows 드라이브)까지 뒤져서 매우 느림 → 범위 지정 또는 `-xdev`

---
