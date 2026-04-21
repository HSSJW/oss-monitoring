# Daily Watch — Claude 루틴 실행 지침

> 매일 오전 9시(KST), 자사 사용 중인 오픈소스와 LLM API의 보안 취약점 ·
> 변동사항 · 지원 종료 공지를 자동 수집·분석하여 GitHub 저장소
> `HSSJW/oss-monitoring`에 보고서로 커밋하고, 긴급 건은 Issue로 생성한다.

---

## 🎯 네가 수행할 일 (한 문장 요약)

저장소의 `config/*.yaml` 두 파일을 읽어 대상 목록을 파악한 뒤, 지정된
3개 채널에서 **전일 이후 신규 항목만** 수집하고, 자사 영향도를 판단하여
보고서를 `reports/` 경로에 커밋하고 긴급 건은 Issue를 생성한다.

---

## 📂 1단계 — 컨텍스트 로드

저장소 `HSSJW/oss-monitoring`의 main 브랜치에서 다음 파일을 읽는다.

1. `config/monitored_oss.yaml` — 관찰 대상 오픈소스 목록
2. `config/monitored_llm_apis.yaml` — 관찰 대상 LLM API 목록

각 파일의 주석에 작성 규칙이 명시되어 있으니, 스키마를 반드시 준수해서 파싱할 것.

> **중요**: `current_version: "확인 필요"` 로 되어 있는 항목은 영향도 판단을
> 건너뛰지 말고, "버전 정보 미등록" 플래그와 함께 "확인 요청" 섹션에 별도 기재한다.

---

## 🔍 2단계 — 데이터 수집 (병렬 수행 가능)

### ─────────────────────────────────────────
### [A] 보안 취약점 수집
### ─────────────────────────────────────────

#### A-1. OSV.dev API (주 채널, 무인증)

```
엔드포인트: POST https://api.osv.dev/v1/query
Content-Type: application/json
인증: 불필요
```

**요청 본문 (라이브러리 1개당):**
```json
{
  "package": {
    "name": "<yaml의 package 값>",
    "ecosystem": "<yaml의 ecosystem 값>"
  }
}
```

**응답 처리:**
- `vulns[]` 배열을 순회
- 각 항목의 `published` 필드가 **전일 00:00 UTC 이후**인 것만 추출
- `severity[].score` (CVSS v3.1 벡터) 파싱 → 숫자 점수 추출
- `database_specific.severity`가 `CRITICAL`이거나 CVSS ≥ 9.0이면 **긴급 플래그**
- `aliases[]`에서 CVE ID, GHSA ID 수집하여 중복 제거

> 모든 모니터링 대상 라이브러리에 대해 호출. 429 응답 시 1초 대기 후 재시도.

#### A-2. GitHub Security Advisory Database (보조)

```
엔드포인트: POST https://api.github.com/graphql
Authorization: Bearer ${GITHUB_TOKEN}
```

**GraphQL 쿼리:**
```graphql
query {
  securityAdvisories(
    first: 100,
    orderBy: {field: PUBLISHED_AT, direction: DESC},
    publishedSince: "<전일 00:00 UTC ISO8601>"
  ) {
    nodes {
      ghsaId
      summary
      description
      severity
      cvss { score vectorString }
      publishedAt
      vulnerabilities(first: 20) {
        nodes {
          package { name, ecosystem }
          firstPatchedVersion { identifier }
          vulnerableVersionRange
        }
      }
      references { url }
    }
  }
}
```

**처리:**
- 응답의 `vulnerabilities.nodes[].package.name`이 `monitored_oss.yaml`의
  `package` 값과 매칭되는 것만 필터
- 매칭된 advisory를 A-1 결과와 병합 (ghsaId 기준 중복 제거)

#### A-3. CISA KEV (실제 악용 취약점)

```
URL: https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json
GET, 인증 불필요
```

**처리:**
- `vulnerabilities[]` 중 `dateAdded`가 전일 이후인 것
- `product` 또는 `vendorProject` 값이 monitored_oss.yaml의 `name`과 매칭
  (대소문자 무시, 부분 일치 허용)
- 매칭되면 **무조건 긴급 플래그** (실제 공격에 쓰인다는 뜻)

### ─────────────────────────────────────────
### [B] 오픈소스 변동사항 수집
### ─────────────────────────────────────────

`monitored_oss.yaml`에서 `github` 값이 null이 아닌 항목만 대상.

#### B-1. GitHub Releases

```
GET https://api.github.com/repos/{owner}/{repo}/releases?per_page=10
Authorization: Bearer ${GITHUB_TOKEN}
```

**처리:**
- 각 릴리즈의 `published_at`이 전일 이후인 것만
- `body` 필드에서 아래 키워드를 **대소문자 무시**로 스캔:
  ```
  security, CVE, vulnerability, vuln, fix, patch,
  breaking change, breaking-change
  ```
- 키워드가 하나라도 있으면 "보안 관련 릴리즈"로 분류
- `tag_name`과 yaml의 `current_version` 비교하여 major/minor/patch 판정
- Pre-release, Draft 릴리즈는 **제외**

#### B-2. GitHub Merged PR

```
GET https://api.github.com/repos/{owner}/{repo}/pulls?state=closed&sort=updated&direction=desc&per_page=30
Authorization: Bearer ${GITHUB_TOKEN}
```

**필터 조건 (모두 만족):**
1. `merged_at`이 전일 이후 (null이면 close만 된 것이므로 제외)
2. 제목 또는 본문에 다음 키워드 중 하나 포함:
   ```
   security, CVE, vulnerability, fix, breaking,
   deprecat, remove, patch
   ```
3. **변경된 파일이 모두 `docs/`, `test/`, `.md`인 경우 제외**
   (단순 문서·테스트 PR은 노이즈)

### ─────────────────────────────────────────
### [C] LLM API 지원 종료 공지 수집
### ─────────────────────────────────────────

#### C-1. 공식 Deprecation 페이지 diff

`monitored_llm_apis.yaml`의 `deprecation_page` URL 각각에 대해:

1. 웹 fetch로 HTML → 본문 텍스트 추출
2. persistent storage에서 `snapshot:{provider}` 키로 전일 스냅샷 로드
3. 현재 텍스트와 diff 수행
4. **신규 추가된 줄**에서 아래 키워드 스캔 (yaml의 `detection_keywords` 참조):
   ```
   영문: deprecat, sunset, retire, shutdown, end of life, EOL, discontinued, removal
   한글: 지원 종료, 서비스 종료, 종료 예정, 사용 중단
   ```
5. 키워드 매칭된 섹션에서 모델명 추출
6. `models_in_use` 목록과 매칭되면 **긴급 플래그**
7. 작업 완료 후 현재 텍스트를 `snapshot:{provider}` 키에 저장 (다음 실행용)

#### C-2. Gmail Connector (선택 사항)

> Gmail 연동이 구성되지 않은 경우 이 단계는 생략. C-1만으로도 대부분의
> 지원 종료 공지는 감지 가능하다.

Gmail 연동이 구성된 경우 다음 검색 실행:

```
from:(noreply@openai.com OR noreply@anthropic.com OR noreply@google.com
      OR no-reply-aws@amazon.com OR azure-noreply@microsoft.com)
subject:(deprecation OR sunset OR retirement OR "end of life"
         OR shutdown OR discontinued)
newer_than:1d
```

---

## 🧠 3단계 — 통합 분석

수집된 모든 항목에 대해 다음 분석을 수행.

### 3-1. 심각도 분류

| 레벨 | 조건 |
|------|------|
| 🔴 **긴급 (CRITICAL)** | CVSS ≥ 9.0 OR CISA KEV 등재 OR `models_in_use` 종료 공지 |
| 🟡 **주의 (WARNING)** | CVSS 7.0~8.9 OR 보안 관련 PR 머지 OR 비사용 모델 종료 공지 |
| 🟢 **참고 (INFO)** | CVSS < 7.0 OR 일반 릴리즈 OR 문서 업데이트 |

### 3-2. 자사 영향도 판별

각 항목에 대해 다음을 확인:
- `current_version`이 취약 버전 범위에 포함되는지
- 포함되면 "영향 있음 (ACTION NEEDED)"
- 포함되지 않으면 "영향 없음 (참고)"
- 버전 정보 없으면 "확인 필요 (UNKNOWN)"

### 3-3. 각 항목 기술 포맷

```markdown
### [심각도 아이콘] {라이브러리/모델명}: {요약}

- **CVE ID / 공지 번호**: CVE-YYYY-XXXXX
- **CVSS 점수**: 9.8 (Critical)
- **영향 범위**: {버전 범위}
- **자사 영향**: {영향 있음 / 없음 / 확인 필요}
- **공식 링크**: {URL}
- **권장 조치**:
  - 업그레이드 대상 버전: {버전}
  - 또는 마이그레이션 가이드: {URL}

**코드 수정 예시** (LLM API 변경의 경우):
​```python
# Before
response = client.chat.completions.create(model="gpt-4-0613", ...)

# After
response = client.chat.completions.create(model="gpt-4.1", ...)
​```
```

---

## 💾 4단계 — 보고서 저장소 커밋

### 4-1. 대상 저장소

```
Repo:   HSSJW/oss-monitoring
Branch: main
```

### 4-2. 파일 경로 및 파일명 규칙

**경로 구조:**
```
reports/{yyyy}/{MM}/[{심각도}]{yyyyMMdd} 일일 보고서.md
```

예시:
```
reports/2026/04/[긴급]20260421 일일 보고서.md
reports/2026/04/[주의]20260420 일일 보고서.md
reports/2026/04/[참고]20260419 일일 보고서.md
```

**심각도 태그 결정 로직 (반드시 이 순서대로):**

```
IF 🔴 긴급 건수 ≥ 1  →  [긴급]
ELIF 🟡 주의 건수 ≥ 1  →  [주의]
ELSE  →  [참고]
```

> 날짜(`yyyyMMdd`)는 한국 시간(KST) 기준 **실행일**.
> 연/월 폴더(`{yyyy}/{MM}`)가 없으면 자동 생성.

### 4-3. GitHub API로 커밋

```
엔드포인트: PUT https://api.github.com/repos/HSSJW/oss-monitoring/contents/{path}
Authorization: Bearer ${GITHUB_TOKEN}
Content-Type: application/json
```

**요청 본문:**
```json
{
  "message": "📊 {yyyy-MM-dd}: 긴급 {N}, 주의 {N}, 참고 {N}",
  "content": "<보고서 본문을 base64로 인코딩한 값>",
  "committer": {
    "name": "Daily Watch Bot",
    "email": "daily-watch@noreply.github.com"
  },
  "branch": "main"
}
```

**동일 파일이 이미 존재하는 경우 (재실행):**
- 먼저 `GET /repos/{owner}/{repo}/contents/{path}`로 기존 파일의 `sha` 조회
- PUT 요청에 `sha` 필드 포함 → 덮어쓰기

### 4-4. 커밋 메시지 규칙

```
형식: 📊 {yyyy-MM-dd}: 긴급 {N}, 주의 {N}, 참고 {N} [ + 주요 이슈 요약 ]

예시:
📊 2026-04-21: 긴급 1, 주의 4, 참고 7 (OpenSSL CVE-2026-3207 등)
📊 2026-04-20: 긴급 0, 주의 2, 참고 5
📊 2026-04-19: 긴급 0, 주의 0, 참고 3 (조용한 날)
```

### 4-5. 보고서 본문 구조

```markdown
# Daily Watch Report — {yyyy-MM-dd (요일)}

**생성 시각**: {yyyy-MM-dd HH:mm KST}
**커버 기간**: {전일 00:00 UTC} ~ {당일 00:00 UTC}

---

## 📊 오늘의 요약
- 🔴 긴급: {N}건
- 🟡 주의: {N}건
- 🟢 참고: {N}건
- ⚠️ 확인 요청: {N}건
- 수집 채널 상태: OSV.dev ✅ · GitHub Advisory ✅ · ...

> 🚨 긴급 건에 대한 Issue가 자동 생성되었습니다: #<issue-번호>

## 🔴 긴급 대응 필요
{항목 없으면 "해당 없음"}
{있으면 3-3 포맷으로 나열, 각 항목에 생성된 Issue 링크 포함}

## 🟡 주의 사항
{동일 포맷}

## 🟢 참고 사항
{동일 포맷, 간결하게}

## ⚠️ 확인 요청
(current_version이 "확인 필요"인 라이브러리 중 이슈 발견 건)
{있으면 나열, 없으면 섹션 생략}

## ⚠️ 시스템 이상
(자가 점검 실패 또는 수집 채널 에러 발생 시)
{있으면 나열, 없으면 섹션 생략}

---
*🤖 이 보고서는 Claude 루틴이 자동 생성했습니다.*
*저장소: [HSSJW/oss-monitoring](https://github.com/HSSJW/oss-monitoring)*
```

---

## 🚨 5단계 — 긴급 건 Issue 자동 생성

### 5-1. 생성 조건

긴급 건 **1건당 Issue 1개**를 생성한다. 다음 조건에 해당하는 항목:
- CVSS ≥ 9.0 신규 CVE
- CISA KEV 신규 등재
- `models_in_use`에 등록된 LLM 모델의 지원 종료 공지

### 5-2. GitHub API로 Issue 생성

```
엔드포인트: POST https://api.github.com/repos/HSSJW/oss-monitoring/issues
Authorization: Bearer ${GITHUB_TOKEN}
Content-Type: application/json
```

**요청 본문 (CVE 케이스):**
```json
{
  "title": "[URGENT][CVE] {라이브러리명} - {한 줄 요약} (CVSS {점수})",
  "body": "<아래 5-3 참조>",
  "labels": ["urgent", "cve", "critical"],
  "assignees": ["HSSJW"]
}
```

**요청 본문 (LLM Deprecation 케이스):**
```json
{
  "title": "[URGENT][Deprecation] {공급사} {모델명} 지원 종료 공지",
  "body": "<아래 5-3 참조>",
  "labels": ["urgent", "deprecation"],
  "assignees": ["HSSJW"]
}
```

### 5-3. Issue 본문 템플릿

```markdown
# 🚨 긴급 대응 필요

| 항목 | 내용 |
|------|------|
| **발견 시각** | {yyyy-MM-dd HH:mm KST} |
| **대상** | {라이브러리/모델명} {버전} |
| **유형** | {CVE / CISA KEV / LLM Deprecation} |
| **심각도** | {CVSS 점수 또는 영향도} |
| **공개 채널** | {출처} |

## 📌 한 줄 요약
{이슈 한 줄 요약}

## 🎯 영향받는 자사 시스템 (추정)
{컨텍스트 기반 추정 목록, 우선순위 P0/P1/P2 분류}

## 📋 권장 조치

### 즉시 (오늘 중)
- [ ] {조치 항목}

### 단기 (이번 주)
- [ ] {조치 항목}

### 중기 (2주 내)
- [ ] {조치 항목}

## 🔗 참고 자료
- 공식 공지: {URL}
- CVE 상세: {URL}
- 관련 보고서: reports/{yyyy}/{MM}/[긴급]{yyyyMMdd} 일일 보고서.md

## 📊 자동 분석 상세
{3-3 포맷의 상세 내용}

---
*🤖 이 Issue는 Claude 루틴이 자동 생성했습니다.*
```

### 5-4. 중복 Issue 방지

- Issue 생성 전 `GET /repos/{owner}/{repo}/issues?labels=cve&state=open`으로
  열린 Issue 목록 조회
- 제목에 동일한 CVE ID가 이미 있으면 **새 Issue 생성하지 않음**
- 대신 기존 Issue에 코멘트 추가: "🔄 {yyyy-MM-dd} 재감지됨"

### 5-5. 보고서와의 연결

- Issue 생성 후 반환된 `html_url`과 `number`를 기록
- 보고서 본문의 해당 항목 상단에 "🎫 Issue: #123" 링크 추가
- 보고서 요약 섹션에 "Issue 자동 생성 {N}건" 명시

---

## ⚙️ 6단계 — 실행 제약 및 재시도

### 실행 시각
```
정규 실행: 매일 09:00 KST (00:00 UTC)
```

### 에러 처리
- API 호출 실패 시: 지수 백오프로 최대 3회 재시도
- 끝까지 실패 시: 해당 채널을 "수집 실패"로 마크하고 보고서에 명시
- **절대 실행 자체를 중단하지 말 것** (다른 채널은 계속 진행)

### Rate Limit
- GitHub API: 인증 상태에서 시간당 5,000회
  - 실행 1회당 예상 사용량: 약 150~200회 (여유 있음)
- OSV.dev: 공식 제한 없음. 정중하게 동시 20개까지
- 웹 fetch: 공급사 문서 페이지는 동시 3개까지

---

## 🔐 7단계 — 필요한 환경변수 / 시크릿

```bash
GITHUB_TOKEN=ghp_xxxxxxxxxxxx   # scope: repo (커밋 + Issue 생성 모두)
```

- Claude 루틴 설정의 Secret 저장소에 등록
- 코드에 하드코딩 금지
- 90일마다 갱신

> **중요**: `public_repo` scope가 아니라 `repo` scope 필요 (Issue 생성 권한 포함).

---

## 🧪 8단계 — 매 실행 시 자가 점검

작업 시작 직전에 다음을 점검하고, 실패 항목은 보고서 상단
"⚠️ 시스템 이상" 섹션에 명시한다.

- `config/monitored_oss.yaml` / `config/monitored_llm_apis.yaml` 파싱 성공 여부
- `GITHUB_TOKEN`으로 `GET /rate_limit` 호출 성공 여부
- 저장소 쓰기 권한 확인 (`GET /repos/HSSJW/oss-monitoring` 응답의 `permissions`)
- persistent storage 읽기·쓰기 동작 여부 (`test:{timestamp}` 키로 확인)

위 중 하나라도 실패하면, 가능한 범위에서만 수집을 진행하고 보고서에
"부분 실행" 상태로 표기한다. **절대 실행 자체를 중단하지 말 것.**

---

## 🔁 9단계 — 중복 보고 방지

동일 CVE / 공지가 매일 반복 보고되지 않도록 다음 로직 적용.

1. 보고서에 포함한 모든 식별자(CVE ID, GHSA ID, 공지 URL)를
   persistent storage에 `reported:{id}` 키로 저장 (TTL 30일)
2. 새 수집 항목이 `reported:*` 키에 이미 존재하면 보고서에서 **skip**
3. 단, CISA KEV 등재 등 상태 변화가 생기면 예외적으로 재보고
   (재보고 시 보고서에 "🔄 상태 업데이트" 표기)
4. 긴급 Issue 중복 방지는 5-4 참조
