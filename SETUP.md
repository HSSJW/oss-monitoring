# 🚀 초기 셋업 가이드

이 저장소를 처음 세팅할 때 한 번만 수행하는 작업들을 정리한 체크리스트입니다.

---

## 1단계 — GitHub 저장소 준비

### 1-1. 이 파일들을 저장소에 Push

로컬에서 이 문서 세트를 받았다면:

```bash
# 이 저장소를 clone
git clone https://github.com/HSSJW/oss-monitoring.git
cd oss-monitoring

# 제공받은 파일 전체를 이 폴더로 복사 후
git add .
git commit -m "🎉 Initial setup: Daily Watch monitoring system"
git push origin main
```

### 1-2. Issue 라벨 생성

저장소 설정 → Labels에서 다음 라벨들을 생성:

| 이름 | 색상 | 설명 |
|------|------|------|
| `urgent` | `#B60205` (진한 빨강) | 긴급 대응 필요 |
| `critical` | `#B60205` | CVSS 9.0 이상 |
| `cve` | `#D93F0B` (주황) | CVE 관련 |
| `deprecation` | `#FBCA04` (노랑) | API/모델 지원 종료 |
| `false-positive` | `#BFDADC` (회색) | 오탐 제보 |
| `missed` | `#BFDADC` | 누락 제보 |

---

## 2단계 — Claude Code 루틴 설정

### 2-1. GitHub Personal Access Token 발급

1. GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)**
2. Generate new token → Generate new token (classic)
3. **Scope**: `repo` (전체 체크 — 커밋 + Issue 생성 모두 가능)
4. Expiration: 90 days 권장
5. 생성된 토큰을 **즉시 복사해서 안전한 곳에 저장**

> ⚠️ `public_repo` scope만 체크하면 Issue 생성 권한이 없어서 실패합니다.
> 반드시 `repo` 전체 scope를 선택하세요.

### 2-2. Claude Code에서 루틴 생성

1. Claude Code 실행 → `/routines` 명령 또는 해당 메뉴 진입
2. New Routine 선택
3. 다음과 같이 설정:

| 항목 | 값 |
|------|-----|
| **Name** | Daily Watch |
| **Schedule** | 매일 09:00 KST (cron: `0 0 * * *` UTC) |
| **Repository** | `HSSJW/oss-monitoring` |
| **Prompt** | `routine_instructions.md` 파일 내용 전체 복사 붙여넣기 |
| **Secrets** | `GITHUB_TOKEN` = 위에서 발급한 토큰 |

### 2-3. 권한 설정

루틴에 다음 권한을 허용:
- ✅ GitHub 저장소 읽기/쓰기
- ✅ 웹 fetch (공급사 공식 문서 페이지용)
- ✅ Persistent storage (스냅샷 저장용)

---

## 3단계 — 설정 파일 커스터마이즈

### 3-1. `config/monitored_oss.yaml` 업데이트

`current_version: "확인 필요"` 로 된 항목들을 실제 사용 중인 버전으로 수정.

확인 방법:
- Java/Maven: `pom.xml` 또는 `build.gradle` 에서 확인
- Python: `requirements.txt`, `pyproject.toml` 에서 확인
- OS 라이브러리: 운영 서버에서 `dpkg -l <패키지명>` (Ubuntu/Debian)
  또는 `rpm -q <패키지명>` (RHEL/CentOS)

### 3-2. `config/monitored_llm_apis.yaml` 업데이트

`models_in_use: ["확인 필요"]` 항목을 실제 사용 중인 모델명으로 수정.

확인 방법:
```bash
# 사내 코드베이스에서 모델명 grep
grep -r "model=" --include="*.py" .
grep -r "model:" --include="*.py" .
```

### 3-3. 변경 사항 커밋

```bash
git add config/
git commit -m "chore: update current_version with production values"
git push origin main
```

---

## 4단계 — 첫 실행 및 검증

### 4-1. 루틴 수동 실행

Claude Code에서 Daily Watch 루틴을 **수동으로 한 번 실행**.

### 4-2. 결과 확인 체크리스트

- [ ] `reports/2026/04/[...]20260421 일일 보고서.md` 파일이 커밋되었는가
- [ ] 커밋 메시지가 `📊 YYYY-MM-DD: 긴급 N, 주의 N, 참고 N` 형식인가
- [ ] 긴급 건이 있었다면 Issue가 생성되었는가
- [ ] Issue에 `urgent` 라벨이 달렸는가
- [ ] 보고서 본문에 시스템 이상 섹션이 비어 있는가 (정상 실행)

### 4-3. 문제 발생 시

- **보고서가 커밋되지 않음** → Claude Code의 루틴 실행 로그 확인
- **"Resource not accessible" 에러** → GITHUB_TOKEN scope 확인 (`repo` 전체)
- **"Not Found" 에러** → 저장소 이름 오타 확인 (`HSSJW/oss-monitoring`)
- **YAML 파싱 실패** → `config/*.yaml` 문법 확인 (들여쓰기, 따옴표)

---

## 5단계 — 일상 운영

### 5-1. 매일 아침 확인 방식

**방법 A: GitHub 웹에서**
- 저장소 방문 → reports/ 폴더 → 오늘 날짜 파일 확인
- 또는 Issues 탭에서 `urgent` 라벨 Issue 확인

**방법 B: GitHub 알림**
- 저장소 Watch 설정: All Activity 또는 Issues only
- 모바일 GitHub 앱으로 푸시 수신

**방법 C: Slack 연동 (선택)**
- GitHub ↔ Slack 공식 앱으로 채널 연결
- `/github subscribe HSSJW/oss-monitoring issues commits`
- 긴급 건은 Slack에 자동 알림

### 5-2. 라이브러리 추가 요청

팀원이 추가 요청:
1. Fork or 브랜치 생성
2. `config/monitored_oss.yaml` 수정
3. PR 제출
4. 리뷰 → 머지
5. 다음날부터 자동 반영

### 5-3. 오탐/누락 제보

- 보고서에 잘못된 내용 → `false-positive` 라벨로 Issue
- 놓친 이슈가 있음 → `missed` 라벨로 Issue
- 수집된 Issue는 추후 지침 개선에 활용

---

## 🎉 완료

모든 단계를 마치면 매일 오전 9시에 자동으로 보고서가 생성되고,
긴급 건은 Issue로 즉시 알림이 갑니다.

문의: Issue 또는 Discussion 이용
