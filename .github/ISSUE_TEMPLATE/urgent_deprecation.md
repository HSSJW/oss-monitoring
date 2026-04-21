---
name: "🚨 긴급 LLM API Deprecation (자동 생성용)"
about: "Claude 루틴이 사용 중인 LLM 모델의 지원 종료 공지 감지 시 자동 생성"
title: "[URGENT][Deprecation] <공급사> <모델명> 지원 종료 공지"
labels: ["urgent", "deprecation"]
assignees: ["HSSJW"]
---

# 🚨 LLM API 지원 종료 — 대응 필요

| 항목 | 내용 |
|------|------|
| **감지 시각** | YYYY-MM-DD HH:mm KST |
| **공급사** | <Anthropic / OpenAI / Google 등> |
| **종료 예정 모델** | `<model-name>` |
| **종료 예정일** | YYYY-MM-DD |
| **남은 기간** | 약 X개월 |
| **권장 대체 모델** | `<new-model-name>` |

---

## 📌 공지 요약

<공지 한 줄 요약>

---

## 🎯 영향받는 자사 서비스

> 컨텍스트 파일 `config/monitored_llm_apis.yaml`의 `models_in_use` 기준으로
> 이 모델이 사용 중으로 등록되어 있습니다.

- <서비스명 1>
- <서비스명 2>

**코드 사용 여부 확인 명령:**
```bash
grep -r "<model-name>" --include="*.py" --include="*.js" --include="*.ts" .
```

---

## 📋 권장 조치

### 즉시 (이번 주)
- [ ] 사내 코드베이스에서 해당 모델명 grep → 실제 사용처 매핑
- [ ] 권장 대체 모델의 API 호환성 확인 (파라미터, 응답 포맷)

### 단기 (2주 내)
- [ ] 스테이징 환경에서 대체 모델로 전환 테스트
- [ ] 출력 품질 회귀 테스트 (기존 응답과 비교)
- [ ] 비용 비교 (토큰 단가 변화)

### 중기 (1개월 내)
- [ ] 프로덕션 환경 단계적 전환 (카나리 → 전체)
- [ ] 모니터링 대시보드에서 지표 추적
- [ ] `config/monitored_llm_apis.yaml`의 `models_in_use` 업데이트

---

## 💻 코드 수정 예시

```python
# Before
response = client.chat.completions.create(
    model="<old-model-name>",
    messages=messages
)

# After
response = client.chat.completions.create(
    model="<new-model-name>",
    messages=messages
)
```

---

## 🔗 참고 자료

- **공식 공지**: <URL>
- **마이그레이션 가이드**: <URL>
- **대체 모델 문서**: <URL>
- **관련 일일 보고서**: `reports/YYYY/MM/[긴급]YYYYMMDD 일일 보고서.md`

---

*🤖 이 Issue는 Claude 루틴이 자동 생성했습니다.*
