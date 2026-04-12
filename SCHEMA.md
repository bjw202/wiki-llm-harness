# Wiki Schema

이 파일은 위키의 구조, 컨벤션, 분류 기준을 정의한다. LLM은 위키 작업 전 이 파일을 먼저 읽는다.

## 디렉토리 구조

```
sources/          ← 원본 자료 (불변, 읽기 전용)
wiki/
  ├── index.md    ← 전체 페이지 카탈로그
  ├── log.md      ← 연산 기록 (시간순)
  ├── .ingest-progress.json  ← 수집 진행 추적 (배치 처리·재개용)
  ├── summaries/  ← 소스별 요약 (허브 페이지, 추출 결과 링크)
  ├── entities/   ← 고유 명사 (인물, 조직, 제품, 장소)
  ├── concepts/   ← 추상 개념 (이론, 기술, 방법론, 원리)
  ├── syntheses/  ← 교차 분석 (여러 페이지를 종합한 통찰)
  └── assets/     ← 이미지, 첨부파일
```

## 페이지 유형 (Karpathy 4유형)

| 유형 | 디렉토리 | 역할 | 크기 제한 |
|------|---------|------|----------|
| **Summary** | summaries/ | 소스 1개의 요약 + 추출된 페이지 링크. 허브 역할. | **30줄 이내** |
| **Entity** | entities/ | 고유 이름이 있는 대상 (인물, 조직, 제품) | 제한 없음 |
| **Concept** | concepts/ | 추상적 아이디어 (이론, 기술, 방법론) | 제한 없음 |
| **Synthesis** | syntheses/ | 여러 엔티티/개념을 교차 분석한 통찰 | 제한 없음 |

### Summary 페이지 규칙 (비대화 방지)

서머리는 **허브 페이지**이지 소스의 복제본이 아니다. 소스를 다시 읽으면 되므로, 서머리에 원문을 옮기는 것은 낭비다.

**서머리에 넣는 것:**
- 소스의 핵심 주장 1~3문장 (executive summary)
- 핵심 발견 3~7개를 한 줄 bullet으로
- 이 소스에서 생성/갱신된 위키 페이지 링크 목록
- 소스의 신뢰도/한계 한 줄 평가

**서머리에 넣지 않는 것:**
- 데이터 테이블, 상세 비교표 → entity/concept 페이지에 넣는다
- 인용문, 상세 논거 → entity/concept 페이지에 넣는다
- 출처 목록 전체 → 소스 원본에 이미 있다

**크기 초과 시:** 30줄을 넘기면 상세 내용을 해당 concept/entity 페이지로 이동하고, 서머리에는 링크만 남긴다.

### Summary 파일명 규칙

소스 파일명을 그대로 쓰지 않는다. 소스 파일명(`01-web-research.md`)은 내용을 반영하지 않아 Obsidian 그래프에서 무의미하다.

**네이밍 형식: `{주제}-{관점/유형}.md`**

- **주제**: 소스가 다루는 핵심 대상 (예: `wiki-llm`, `digital-twin`, `manufacturing-ai`)
- **관점/유형**: 소스의 성격을 구분하는 수식어 (예: `web-research`, `academic-review`, `community-voices`, `architecture-design`)

관점/유형은 소스의 조사 방법이나 저자 시각을 반영한다. concept/entity 페이지는 "무엇에 대한 지식"이고, summary는 "누가/어떻게 조사한 결과"이므로 자연스럽게 이름이 구분된다.

**충돌 방지**: 이름을 정한 후 `wiki/` 전체(summaries/, concepts/, entities/, syntheses/)에서 Glob으로 동일 이름이 있는지 확인한다. 충돌하면 관점/유형을 더 구체적으로 변경한다.

## Frontmatter 규칙

### 필수 필드

**entities / concepts / syntheses:**

```yaml
---
title: "페이지 제목"
tags: [카테고리태그, 주제태그]
sources: ["출처파일명1", "출처파일명2"]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

**summaries** (다른 페이지와 다름 — `sources` 배열 대신 `source-file` 단일 값):

```yaml
---
title: "소스 제목"
tags: [summary, 주제태그]
source-file: "원본파일명"
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

이 차이가 있는 이유: summary는 소스 1개에 대응하므로 단일 값이고, entity/concept은 여러 소스에서 정보가 축적되므로 배열이다.

### 카테고리 태그 (필수, 하나 이상)

- `summary` — summaries/ 하위 페이지
- `entity` — entities/ 하위 페이지
- `concept` — concepts/ 하위 페이지
- `synthesis` — syntheses/ 하위 페이지

### 선택 필드

- `aliases`: 대안 이름 배열 (Obsidian 검색용)
- `needs-review`: 검토 필요 여부 (boolean)
- `generated-from`: 생성 계기 ("ingest" 또는 "query")
- `related`: 관련 페이지 배열

## 파일명 컨벤션

- 소문자, 단어 구분은 하이픈(`-`)
- `.md` 확장자
- 한글 제목은 한글 파일명 허용
- 공백·특수문자 금지
- 길이 제한: 50자 이내
- **summaries 파일명**: `{주제}-{관점/유형}.md` 형식. 소스 파일명을 그대로 쓰지 않는다. 상세 규칙은 위 "Summary 파일명 규칙" 참조.

**충돌 해결**: 같은 파일명이 이미 존재하면 기존 파일의 내용을 확인한다. 같은 주제면 갱신하고, 다른 주제면 `-2`, `-3` 접미사를 붙인다.

## 교차 참조 규칙

- `[[폴더/파일명]]` 형식 (확장자 생략)
- 관련성이 명확한 경우에만 링크한다
- 가능하면 양방향 링크를 유지한다
- **sources/ 파일은 `[[wiki-link]]`로 참조하지 않는다** — Obsidian vault 밖이므로 `` `sources/파일명` `` 코드 형식을 사용한다

## 출처 표기

- 본문 내 주장에는 출처를 명시한다
- 형식: 문장 끝에 `(출처: `sources/파일명`)` — wiki-link가 아닌 코드 형식

## 진행 추적 (Ingest)

`wiki/.ingest-progress.json`은 대량 수집 시 배치 처리 상태를 기록한다. 이 파일이 존재하면 이전 세션에서 중단된 작업이 있으므로, 큐에 남은 파일부터 이어서 처리한다.

## 연산 기록 형식 (log.md)

```markdown
## [YYYY-MM-DD] {연산} | {설명}
- {상세 내용}
```

연산 종류: `ingest`, `query`, `lint`
