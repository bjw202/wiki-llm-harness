---
name: wiki-ingest
description: "소스 자료를 위키에 수집(ingest)하는 스킬. 파일/URL을 읽고 소스별 요약(summary) + 개념/엔티티/통합 페이지를 생성·갱신하며 index와 log를 업데이트한다. 100개 이상의 파일도 배치 단위로 안전하게 처리하며, 중단 후 재개가 가능하다. '위키에 추가', '소스 수집', 'ingest', '이 문서를 위키에', '새 자료 등록', '소스 폴더 전체 수집' 등의 요청 시 반드시 이 스킬을 사용한다."
---

# Wiki Ingest

소스를 읽고, 요약(summary) + 개념/엔티티 페이지를 생성·갱신하며, 교차 참조와 인덱스를 유지하는 수집 스킬.

## 핵심 원칙

- **원본 불변**: `sources/` 파일은 읽기만 한다
- **요약은 허브**: summary는 소스의 복제본이 아니라 "이 소스에서 무엇을 추출했는지" 보여주는 허브 페이지다. 상세 내용은 concept/entity 페이지에 둔다
- **중단 안전**: `wiki/.ingest-progress.json`으로 재개 가능
- **네이밍 충돌 방지**: 기존 파일과 충돌 시 내용 비교 후 갱신 또는 접미사 부여

## 워크플로우

### 1. 진행 상태 확인 (재개 지원)

`wiki/.ingest-progress.json`을 읽는다. 있으면 `queue`에 남은 파일부터 이어서 처리한다.

### 2. 소스 스캔 및 큐 구성

`sources/` 디렉토리를 Glob으로 스캔. 이미 `completed`에 있는 파일은 제외하고 새 파일만 `queue`에 넣는다.

### 3. 배치 분할

**컨텍스트 오버플로 방지가 핵심이다.**

- 기본: **5개 파일/배치** (큰 파일: 3개, 작은 파일: 10개)
- 한 배치 처리 후: progress 저장 → 사용자에게 보고 → 남은 큐 있으면 계속 여부 질문

### 4. 소스 분석 및 추출 (배치 내 파일별)

각 소스를 읽고 추출한다:
- **엔티티**: 인물, 조직, 제품, 장소 등 고유 명사
- **개념**: 이론, 기술, 방법론, 원리 등
- **사실**: 데이터, 통계, 날짜, 인용문
- **관계**: 엔티티-엔티티, 엔티티-개념 간 연결

### 5. Summary 페이지 생성 (소스 1개 = Summary 1개)

**파일명: `{주제}-{관점/유형}.md`** — 소스 파일명을 그대로 쓰지 않는다. Obsidian 그래프에서 의미 있는 이름이어야 한다.

파일명 결정 절차:
1. 소스의 핵심 주제를 파악한다 (예: wiki-llm, digital-twin)
2. 소스의 성격/관점을 파악한다 (예: web-research, academic-review, architecture-design)
3. `{주제}-{관점}.md`로 조합한다 (예: `digital-twin-community-voices.md`)
4. `wiki/` 전체에서 동일 이름이 있는지 Glob으로 확인한다. 충돌 시 관점을 더 구체화한다

concept 페이지는 "무엇에 대한 지식"이고, summary는 "누가/어떻게 조사한 결과"이므로 자연스럽게 구분된다.

**30줄 이내 엄수.** 서머리가 비대해지면 서머리끼리 중복·모순이 발생하고, query 시 서머리만 읽고 concept 페이지를 놓치는 문제가 생긴다.

```markdown
---
title: "소스 제목"
tags: [summary, 주제태그]
source-file: "원본파일명"
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# 소스 제목

{핵심 주장 1~3문장}

## 핵심 발견

- 발견1 한 줄
- 발견2 한 줄
- 발견3 한 줄

## 추출된 페이지

- [[concepts/xxx]] — 생성/갱신
- [[entities/yyy]] — 생성/갱신

## 소스 평가

신뢰도: ★★☆ | 한계: {한 줄}
```

**넣지 않는 것**: 데이터 테이블, 상세 비교표, 인용문, 출처 목록 전체. 이것들은 concept/entity 페이지에 넣는다.

### 6. Entity/Concept 페이지 생성/갱신

**파일명 결정**: 후보 생성 → Glob 충돌 검사 → 같은 주제면 갱신, 다른 주제면 접미사.

**새 페이지**: frontmatter + 본문 + 출처 (SCHEMA.md 규칙 참조)
**기존 페이지 갱신**: 기존 내용 보존, `updated` 갱신, `sources` 배열에 추가. 모순 시 `> [!warning]` 콜아웃.

**출처 표기**: `(출처: `sources/파일명`)` — wiki-link가 아닌 코드 형식. Obsidian vault 밖의 파일을 `[[]]`로 참조하면 그래프에 유령 노드가 생긴다.

### 7. 교차 참조

- 새 페이지에서 기존 페이지로, 기존 페이지에서 새 페이지로 양방향 `[[링크]]`
- summary → concept/entity 방향 링크는 필수
- concept/entity → summary 방향 링크는 불필요 (frontmatter의 sources 필드로 충분)

### 8. 인덱스·로그 갱신

**index.md**: Summaries, Entities, Concepts, Syntheses 4개 섹션에 새 페이지 삽입.

**log.md**: 배치 단위로 기록:
```markdown
## [YYYY-MM-DD] ingest | 배치 N (파일 M개)
- 소스: file-a.pdf, file-b.md
- 요약: [[summaries/file-a]], [[summaries/file-b]]
- 생성: [[concepts/xxx]], [[entities/yyy]]
- 갱신: [[concepts/zzz]]
```

### 9. 진행 상태 저장

배치 처리 후 `wiki/.ingest-progress.json` 갱신. 완료 시 `queue`가 빈 배열.

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 소스 읽기 실패 | `failed`에 기록, 나머지 계속 |
| 서머리 30줄 초과 | 상세 내용을 concept/entity로 이동, 서머리에는 링크만 |
| 파일명 충돌 | 내용 비교 → 같으면 갱신, 다르면 접미사 |
| 기존 페이지와 모순 | 두 출처 병기, `> [!warning]` 콜아웃 |
| 카테고리 분류 모호 | `concepts/`에 배치, `needs-review: true` |
