---
name: wiki-lint
description: "위키의 건강 상태를 점검(lint)하는 스킬. 모순, 고아 페이지, 깨진 링크, 누락된 교차 참조, 정보 공백, frontmatter 오류, summary 크기 초과를 찾아 보고하고 안전한 수정을 적용한다. 'lint', '위키 점검', '건강 체크', '위키 상태', '정합성 검사' 등의 요청 시 사용한다."
---

# Wiki Lint

위키 전체를 순회하며 구조적·내용적 문제를 찾아 보고하고, 안전한 수정을 적용하는 건강 점검 스킬.

## 워크플로우

### 1. 규칙 로드

`SCHEMA.md`를 읽어 위키의 규칙(frontmatter 필수 필드, 카테고리 정의, 파일명 컨벤션, summary 크기 제한)을 확인한다.

### 2. 페이지 수집

Glob으로 `wiki/**/*.md`를 검색하여 모든 위키 페이지를 수집한다. `index.md`와 `log.md`는 점검 대상에서 제외하되 정합성 비교에는 사용한다.

### 3. 구조 점검

각 페이지에 대해:

- **frontmatter 유효성**: 필수 필드 존재 여부
  - 모든 페이지: `title`, `tags`, `created`, `updated`
  - summaries: 위 필드 + `source-file` (다른 페이지의 `sources` 배열과 다름)
  - entities/concepts/syntheses: 위 필드 + `sources` 배열
- **파일명 컨벤션**: 소문자, 하이픈 구분 준수 여부
- **카테고리 일치**: 파일 위치(summaries/entities/concepts/syntheses)와 tags의 카테고리 태그 일치 여부

### 4. Summary 전용 점검

summaries/ 하위 페이지에 대해 추가로:

- **크기 제한**: 본문이 30줄을 초과하는지 확인. 초과 시 warning 보고
- **추출 링크 존재**: "추출된 페이지" 섹션에 최소 1개 이상의 `[[링크]]`가 있는지
- **소스 파일 존재**: `source-file` frontmatter가 가리키는 파일이 `sources/`에 실제로 있는지

### 5. 링크 점검

모든 `[[wiki-link]]`를 추출하고:

- **깨진 링크**: 대상 파일이 존재하지 않는 링크
- **유령 노드**: `[[sources/...]]` 형태의 wiki-link (vault 밖 파일 참조 — `` `sources/...` `` 코드 형식이어야 함)
- **고아 페이지**: 어디서도 참조되지 않는 페이지 (index.md 제외)
- **양방향 확인**: A→B 참조 시 B→A도 참조하는지 (권장, 필수 아님)

### 6. 내용 점검

페이지 간 교차 비교:

- **모순 탐지**: 같은 주제에 대해 다른 페이지에서 상충하는 주장
- **중복 탐지**: 같은 내용을 다루는 별도의 페이지
- **출처 누락**: 주장에 출처 표기가 없는 문단
- **재분류 후보**: frontmatter에 `needs-review: true`가 있는 페이지 — ingest 시 entity/concept 분류가 모호해 일단 concepts/에 배치된 페이지. 보고서에 별도 섹션으로 리스트업하여 사용자가 검토할 기회를 준다. 검토 후 올바른 위치로 이동하거나 needs-review 플래그를 제거한다.

### 7. 인덱스 정합성

- `wiki/index.md`에 있지만 실제 파일이 없는 항목
- 실제 파일이 있지만 `wiki/index.md`에 없는 항목
- index.md의 4개 섹션(Summaries/Entities/Concepts/Syntheses) 존재 여부

### 8. 보고 및 수정

**자동 수정 (직접 적용):**
- index.md에 누락된 페이지 추가
- 깨진 링크 제거
- frontmatter 필수 필드 추가 (빈 값)
- `[[sources/...]]` wiki-link를 `` `sources/...` `` 코드 형식으로 변환

**보고만 (수동 검토 필요):**
- 모순된 내용
- 중복 페이지
- 카테고리 부적합
- summary 30줄 초과
- `needs-review: true` 플래그 (entity/concept 재분류 필요 후보)

### 9. 로그 기록

`.wiki-log.md`에 lint 기록을 추가한다:

```markdown
## [YYYY-MM-DD] lint | 전체 점검
- 총 페이지: N (summaries: N, entities: N, concepts: N, syntheses: N)
- 문제 발견: N (critical: N, warning: N, info: N)
- 자동 수정: N
- 상세: {각 문제 요약}
```

## 심각도 기준

| 심각도 | 기준 | 예시 |
|--------|------|------|
| critical | 정보 정확성에 영향 | 페이지 간 모순, 출처 없는 핵심 주장 |
| warning | 탐색성에 영향 | 깨진 링크, 고아 페이지, index 불일치, summary 30줄 초과, `[[sources/]]` 유령 노드 |
| info | 개선 권장 | 양방향 링크 누락, frontmatter 선택 필드 누락 |
