# Daily Watch Report — 2026-04-21 (월)

**생성 시각**: 2026-04-21 09:00 KST
**커버 기간**: 2026-04-20 00:00 UTC ~ 2026-04-21 00:00 UTC

---

## 📊 오늘의 요약

| 구분 | 건수 |
|------|------|
| 🔴 긴급 | **1건** |
| 🟡 주의 | 4건 |
| 🟢 참고 | 7건 |
| ⚠️ 확인 요청 | 2건 |
| **합계** | **14건** |

**수집 채널 상태**: OSV.dev ✅ · GitHub Advisory ✅ · CISA KEV ✅ · GitHub Releases ✅ · GitHub PR ✅ · Anthropic 문서 ✅ · OpenAI 문서 ✅ · Google 문서 ✅ · Gmail ✅

> 🚨 긴급 건에 대한 별도 [URGENT] 메일이 09:02 KST에 발송되었습니다.

---

## 🔴 긴급 대응 필요

### 🔴 OpenSSL: Use-after-free in X.509 certificate verification

- **CVE ID**: CVE-2026-3207
- **CVSS 점수**: 9.8 (Critical)
- **영향 범위**: OpenSSL 3.0.0 ~ 3.2.5, 3.3.0 ~ 3.3.4
- **자사 영향**: ✅ **영향 있음** (현재 사용: 3.2.3)
- **공개 일시**: 2026-04-20 14:32 UTC
- **출처**: OSV.dev + GitHub Advisory (GHSA-xxxx-yyyy-zzzz)
- **공식 링크**: https://www.openssl.org/news/secadv/20260420.txt

**요약**
인증서 검증 과정에서 use-after-free 취약점이 발견되었습니다. 원격 공격자가 조작된 인증서를 제공하면 임의 코드 실행이 가능합니다. TLS 서버·클라이언트 모두 영향받습니다.

**권장 조치 (우선순위 순)**
1. **즉시**: OpenSSL 3.2.6 또는 3.3.5로 긴급 업그레이드
2. **단기**: TLS 종료가 걸린 모든 서비스(API Gateway, Reverse Proxy) 재시작
3. **중기**: 인증서 검증 로직이 있는 내부 서비스 전수 조사

**영향받는 자사 시스템 (추정)**
- API Gateway (Nginx + OpenSSL 3.2.3)
- 내부 mTLS 통신 (Service Mesh)
- 메일 서버 TLS 연결

---

## 🟡 주의 사항

### 🟡 Django: SQL injection in QuerySet.filter() with specific annotations

- **CVE ID**: CVE-2026-3188
- **CVSS 점수**: 8.1 (High)
- **영향 범위**: Django 4.2.0 ~ 4.2.18, 5.0.0 ~ 5.0.10, 5.1.0 ~ 5.1.4
- **자사 영향**: ⚠️ **확인 필요** (current_version 미등록)
- **공개 일시**: 2026-04-20 18:00 UTC
- **공식 링크**: https://www.djangoproject.com/weblog/2026/apr/20/security-releases/

**요약**
`QuerySet.filter()`에 특정 형태의 annotation과 user input이 결합될 때 SQL injection이 가능합니다. 패치 버전: 4.2.19, 5.0.11, 5.1.5.

**권장 조치**
- 운영 중인 Django 버전 확인 → 해당되면 즉시 업그레이드
- 업그레이드 불가 시 annotation에 사용자 입력이 들어가는 쿼리 전수 점검

---

### 🟡 libxml2: Heap overflow in XPath evaluation

- **CVE ID**: CVE-2026-2941
- **CVSS 점수**: 7.5 (High)
- **영향 범위**: libxml2 2.11.0 ~ 2.12.9
- **자사 영향**: ⚠️ **확인 필요** (current_version 미등록)
- **출처**: OSV.dev (Debian:12 ecosystem)

**요약**
복잡한 XPath 표현식 평가 중 힙 오버플로우 발생. DoS 및 잠재적 RCE 가능성.

**권장 조치**
- libxml2 2.12.10 이상으로 업그레이드
- XML 파싱을 외부 입력에 노출하는 서비스 우선 대응

---

### 🟡 Apache Tomcat: Security-related PR merged (#12847)

- **저장소**: apache/tomcat
- **PR 제목**: "Fix request smuggling vulnerability in HTTP/2 parser"
- **머지 일시**: 2026-04-20 22:15 UTC
- **변경 규모**: +127 / -84, 파일 8개
- **관련 CVE**: 아직 미할당 (다음 릴리즈에 포함 예상)
- **링크**: https://github.com/apache/tomcat/pull/12847

**분석**
HTTP/2 프로토콜 파서의 request smuggling 취약점 수정. 정식 릴리즈 전 사전 인지 차원.

**권장 조치**
- 다음 Tomcat 릴리즈(10.1.x) 공지 주시
- HTTP/2가 활성화된 서비스 목록 사전 정리

---

### 🟡 GPT-4.1 Deprecation 공지

- **공급사**: OpenAI
- **종료 예정 모델**: `gpt-4.1`, `gpt-4.1-mini`
- **자사 영향**: ⚠️ **확인 필요** (models_in_use 미등록)
- **공지 일시**: 2026-04-20 16:00 UTC
- **종료 예정일**: 2026-10-20 (6개월 유예)
- **권장 대체**: `gpt-5.4`, `gpt-5.4-mini`
- **출처**: OpenAI 공식 Deprecation 페이지 diff 감지
- **링크**: https://platform.openai.com/docs/deprecations

**코드 수정 예시**
```python
# Before
response = client.chat.completions.create(
    model="gpt-4.1",
    messages=messages
)

# After
response = client.chat.completions.create(
    model="gpt-5.4",
    messages=messages
)
```

**권장 조치**
- 사내 코드베이스에서 `gpt-4.1` 문자열 grep → 사용 여부 확인
- 사용 중이면 스테이징에서 `gpt-5.4` 호환성 테스트 시작

---

## 🟢 참고 사항

### 🟢 PostgreSQL 16.3 릴리즈
- **저장소**: postgres/postgres
- **릴리즈 노트**: 버그 수정 중심. 보안 이슈 없음
- **주요 변경**: VACUUM 성능 개선, 파티션 테이블 통계 정확도 향상

### 🟢 FastAPI 0.115.0 릴리즈
- **저장소**: fastapi/fastapi
- **주요 변경**: WebSocket 의존성 주입 개선, Pydantic 2.10 호환성
- **Breaking change**: 없음

### 🟢 curl 8.8.0 릴리즈
- **저장소**: curl/curl
- **주요 변경**: HTTP/3 QUIC 안정성 개선
- **보안 키워드**: 없음

### 🟢 Anthropic Claude API Changelog
- Messages API에 `cache_control` TTL 옵션 추가 (지원 종료 아님, 기능 추가)

### 🟢 Jetty 12.0.12 릴리즈
- **저장소**: jetty/jetty.project
- **주요 변경**: 버그 수정, 성능 튜닝

### 🟢 zstd 1.5.6 릴리즈
- **저장소**: facebook/zstd
- **주요 변경**: 압축 레벨 튜닝, ARM64 최적화

### 🟢 Flask 3.0.4 릴리즈
- **저장소**: pallets/flask
- **주요 변경**: 문서 개선, 의존성 업데이트

---

## ⚠️ 확인 요청

아래 항목은 이슈가 발견되었으나 `current_version` 또는 `models_in_use`가
미등록 상태라 영향도 판단을 할 수 없었습니다.

### 1. Django 버전 확인 필요
- 위 🟡 CVE-2026-3188 관련
- `monitored_oss.yaml`에 `current_version: "확인 필요"`로 등록됨
- **요청**: 실제 운영 중인 Django 버전을 운영팀에 확인 후 YAML 업데이트

### 2. OpenAI 사용 모델 확인 필요
- 위 🟡 GPT-4.1 deprecation 관련
- `monitored_llm_apis.yaml`에 `models_in_use: ["확인 필요"]`로 등록됨
- **요청**: 실제 사용 중인 OpenAI 모델명을 코드베이스에서 확인 후 YAML 업데이트

---

## 📈 주간 트렌드 (최근 7일 누적)

- 긴급: 2건 (이번 주) vs 0건 (지난 주)
- 주의: 18건 vs 12건
- 참고: 47건 vs 52건
- **주목**: OpenSSL 관련 이슈가 최근 증가 추세 (4건 / 주)

---

## 🔧 시스템 상태

- 다음 실행 예정: 2026-04-22 09:00 KST
- 루틴 버전: v1.2.0
- 컨텍스트 파일 마지막 수정: 2026-04-15 (6일 전)
- persistent storage 사용량: 2.4 MB / 100 MB

---

*이 보고서는 Claude 루틴이 자동 생성했습니다.*
*오탐·누락 제보: security-team@company.com*
*라이브러리 추가 요청: [GitHub Repo PR](https://github.com/company/daily-watch/pulls)*
