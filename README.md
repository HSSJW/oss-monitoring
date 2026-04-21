# 🛡️ OSS Monitoring — Daily Watch

> Claude 루틴 기반 오픈소스 보안 취약점 · LLM API 지원 종료 **일일 자동 모니터링** 시스템

매일 오전 9시(KST), 자사 사용 중인 오픈소스 라이브러리와 LLM API의 보안 취약점,
변동사항, 지원 종료 공지를 자동 수집·분석하여 이 저장소에 보고서로 커밋합니다.
긴급 건(CVSS ≥ 9.0 등) 발생 시 **GitHub Issue를 자동 생성**하여 담당자에게 알립니다.

[![Reports](https://img.shields.io/badge/reports-daily-blue)](./reports)
[![Issues](https://img.shields.io/badge/urgent-auto--issue-red)](../../issues)

---

## 📂 저장소 구조

```
oss-monitoring/
├── README.md                         ← 이 문서
├── routine_instructions.md           ← Claude 루틴 실행 지침
│
├── config/                           ← 모니터링 대상 정의 (팀이 PR로 관리)
│   ├── monitored_oss.yaml            ← 관찰 오픈소스 목록 (60+개)
│   └── monitored_llm_apis.yaml       ← 관찰 LLM API 공급사
│
├── reports/                          ← 자동 생성되는 보고서 (루틴이 커밋)
│   └── {yyyy}/{MM}/
│       └── [{심각도}]{yyyyMMdd} 일일 보고서.md
│
├── samples/                          ← 보고서 예시 (참고용)
│   ├── daily_report_sample.md
│   ├── daily_report_quiet_day_sample.md
│   └── urgent_alert_sample.md
│
└── .github/
    └── ISSUE_TEMPLATE/               ← 긴급 Issue 템플릿
        ├── urgent_cve.md
        └── urgent_deprecation.md
```

---

## 📌 보고서 파일명 규칙

그날의 **최고 심각도** 기준으로 파일명이 결정됩니다.

| 파일명 예시 | 조건 |
|------------|------|
| `[긴급]20260421 일일 보고서.md` | 🔴 긴급 1건 이상 |
| `[주의]20260420 일일 보고서.md` | 🔴 없음 + 🟡 주의 1건 이상 |
| `[참고]20260419 일일 보고서.md` | 🔴·🟡 없음 (조용한 날) |

폴더에서 파일명만 봐도 오늘 긴급 건 여부를 바로 알 수 있습니다.

---

## 🚨 긴급 Issue 자동 생성

다음 조건 중 하나라도 해당되면 루틴이 **GitHub Issue를 자동 생성**합니다.

- CVSS ≥ 9.0 신규 CVE
- CISA KEV(실제 악용 목록) 신규 등재
- `models_in_use`에 등록된 LLM 모델의 지원 종료 공지

Issue 라벨: `urgent`, `cve` 또는 `deprecation`, 심각도별 라벨
자동 멘션: `@HSSJW` (수정 가능)

---

## 🚀 초기 셋업 가이드

### 1. 이 저장소 Clone 또는 Fork
```bash
git clone https://github.com/HSSJW/oss-monitoring.git
cd oss-monitoring
```

### 2. `config/*.yaml` 커스터마이즈
- `config/monitored_oss.yaml`의 `current_version: "확인 필요"` 항목들을
  실제 운영 중인 버전으로 업데이트
- `config/monitored_llm_apis.yaml`의 `models_in_use` 항목 역시
  실제 사용 중인 모델명으로 업데이트

### 3. Claude 루틴 생성
Claude Code에서 새 루틴을 생성하고 다음을 설정:

- **지침(Prompt)**: `routine_instructions.md` 내용 전체 복사
- **저장소 연결**: `HSSJW/oss-monitoring`
- **스케줄**: 매일 09:00 KST
- **필요 권한**:
  - `repo` scope의 GitHub Token (커밋 + Issue 생성용)

### 4. GitHub Token 발급 및 등록
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 발급 scope: `repo` (전체 - 커밋 + Issue 생성)
3. Claude 루틴의 Secret 저장소에 `GITHUB_TOKEN`으로 등록

### 5. 첫 실행 테스트
- 루틴을 수동 실행하여 `reports/` 폴더에 파일이 커밋되는지 확인
- 아래 "첫 실행 전 검증 체크리스트" 수행

---

## 🤖 루틴이 하는 일 (한눈에 보기)

```
매일 09:00 KST
    ↓
① config/*.yaml 읽기
    ↓
② 데이터 수집 (병렬)
   ├─ OSV.dev API         (보안 취약점)
   ├─ GitHub Advisory DB  (보안 취약점 보조)
   ├─ CISA KEV            (실제 악용 CVE)
   ├─ GitHub Releases     (릴리즈 감지)
   ├─ GitHub PR           (보안 관련 머지)
   └─ 공식 Deprecation 문서 (LLM API 종료)
    ↓
③ Claude 통합 분석
   - 심각도 분류 (🔴/🟡/🟢)
   - 자사 영향도 판별
   - 권장 조치 + 코드 예시 생성
    ↓
④ 결과 배포
   ├─ reports/에 .md 파일 커밋
   └─ 긴급 건 발생 시 Issue 자동 생성
```

---

## 🛠️ 라이브러리 추가 / 제거

### 새 라이브러리 추가
```yaml
# config/monitored_oss.yaml 끝에 추가

  - name: "새 라이브러리"
    ecosystem: "PyPI"           # OSV.dev 기준 값 (대소문자 구분)
    package: "package-name"
    github: "owner/repo"         # 없으면 null
    current_version: "1.2.3"
    category: "Web/WAS"          # 또는 DBMS, OS
    priority: "medium"           # high / medium / low
    note: ""
```

PR 생성 → 리뷰 → 머지 → 다음 실행부터 자동 반영됩니다.

### 라이브러리 제거
해당 블록 삭제 후 PR 제출.

---

## 📊 수집 채널 요약

| 유형 | 채널 | 인증 | 용도 |
|------|------|------|------|
| 취약점 | OSV.dev API | 불필요 | 주 채널 |
| 취약점 | GitHub Advisory GraphQL | Token 필수 | 보조 |
| 취약점 | CISA KEV JSON | 불필요 | 실제 악용 CVE |
| 변동 | GitHub Releases API | Token 필수 | 신규 릴리즈 |
| 변동 | GitHub PR API | Token 필수 | 보안 관련 머지 |
| LLM API | 공급사 공식 문서 fetch | 불필요 | 지원 종료 공지 |
| LLM API | Gmail Connector (선택) | 계정 필수 | 공지 메일 (옵션) |

---

## 🚨 심각도 분류

| 레벨 | 기준 | 조치 |
|------|------|------|
| 🔴 긴급 | CVSS ≥ 9.0 / CISA KEV 등재 / 사용 중 LLM 모델 종료 | **Issue 자동 생성 + 보고서** |
| 🟡 주의 | CVSS 7.0~8.9 / 보안 PR 머지 / 비사용 모델 종료 | 보고서에만 기록 |
| 🟢 참고 | CVSS < 7.0 / 일반 릴리즈 | 보고서에만 기록 |

---

## ✅ 첫 실행 전 검증 체크리스트

- [ ] `config/monitored_oss.yaml`의 ecosystem 값이 OSV.dev 공식 값과 일치
- [ ] `github` 필드의 저장소가 실제로 존재하고 공개
- [ ] `config/monitored_llm_apis.yaml`의 deprecation_page URL이 실제로 fetch 가능
- [ ] GITHUB_TOKEN으로 인증 호출 성공 (`GET /rate_limit`)
- [ ] 테스트 커밋 1건이 정상 생성되는지 확인
- [ ] 테스트 Issue 1건이 정상 생성되는지 확인

---

## 📝 운영 튜닝 가이드

### 첫 1~2주 (안정화)
- 긴급 임계값 CVSS **8.0**으로 낮춰 감도 확인
- Issue 자동 멘션은 본인으로만 설정
- 매일 커밋된 보고서를 수동 검토하여 오탐·누락 파악

### 2주 후 (정착)
- 긴급 임계값 CVSS **9.0**으로 환원
- 멘션 대상을 팀 리스트로 확장
- 오탐 많은 PR 키워드 추가 제외 (예: `typo`, `chore`, `refactor`)

### 1개월 후 (확장)
- DBMS → OS 라이브러리 단계적 추가
- 경량 실행 (4시간 주기 CISA KEV 체크) 활성화 검토

---

## 🐛 문제 해결

| 증상 | 확인 사항 |
|------|---------|
| 3일 연속 실행 실패 | GITHUB_TOKEN 만료 여부 확인 (90일 주기) |
| 커밋은 되는데 Issue가 안 생김 | Token의 `repo` scope 확인 |
| 보고서가 비어 있음 | YAML 문법 오류 가능성 → 파싱 테스트 |
| 동일 CVE 반복 보고 | 루틴의 persistent storage `reported:*` 키 점검 |

---

## 📞 문의 및 기여

- **오탐·누락 제보**: Issue 생성 (라벨 `false-positive` 또는 `missed`)
- **기능 제안**: Discussion 또는 Issue
- **라이브러리 추가 요청**: PR 제출

---

*🤖 이 저장소의 대부분 콘텐츠는 Claude 루틴이 자동 생성합니다.*
