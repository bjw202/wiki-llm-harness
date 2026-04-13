# wiki-llm-harness

**Andrej Karpathy의 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 패턴을 Claude Code 스킬로 구현한 지식 관리 하네스.**

Obsidian을 편집기로 쓰고, Claude가 위키를 유지·관리한다. 인간은 소스를 큐레이션하고 질문만 하면, LLM이 읽기·요약·교차참조·정합성 검증을 전담한다.

---

## 목차

1. [Karpathy의 철학](#karpathy의-철학)
2. [아키텍처](#아키텍처)
3. [설계 결정의 배경 — 실전에서 부딪힌 문제들](#설계-결정의-배경--실전에서-부딪힌-문제들)
4. [설치 및 설정](#설치-및-설정)
5. [사용법](#사용법)
6. [프로젝트 구조](#프로젝트-구조)

---

## Karpathy의 철학

Karpathy의 gist를 관통하는 한 문장:

> "지식 기반 유지보수의 지루한 부분은 읽기나 사고가 아니라 **장부 정리(bookkeeping)**다."

기존 RAG는 질의마다 원본을 재발견한다. 매번 인터프리터가 돌아가는 셈이다. wiki-llm은 다르게 접근한다 — 소스를 **한 번 컴파일**해서 영속적 마크다운 위키로 저장하고, 질의 시에는 이 위키만 읽는다. 컴파일러와 인터프리터의 차이다.

### 3계층 아키텍처

| 계층 | 역할 | 소유자 |
|------|------|--------|
| **Raw Sources** | 불변 원본 자료 (논문, 기사, 메모) | 인간이 큐레이션 |
| **Wiki** | LLM이 유지하는 마크다운 파일 집합 | LLM이 관리 |
| **Schema** | 위키 규칙·컨벤션 정의 | 인간이 설계, LLM이 준수 |

### 3대 오퍼레이션

- **Ingest** — 새 소스를 처리해 10~15개 관련 페이지를 동시에 갱신. 교차 참조가 자동 유지됨
- **Query** — 위키 페이지를 검색·종합해 출처 기반 답변 생성. 고가치 답변은 위키로 역환류
- **Lint** — 모순, 고아 페이지, 깨진 링크, 오래된 주장 탐지. 위키의 건강 유지

### 4가지 페이지 유형 (Karpathy 원안)

1. **Summary** — 소스 1개당 1개의 허브 페이지
2. **Entity** — 고유 명사 (인물, 조직, 제품)
3. **Concept** — 추상 개념 (이론, 기술, 방법론)
4. **Synthesis** — 교차 분석·통합 정리

---

## 아키텍처

Claude Code의 **스킬(Skill)**로 구현했다. 에이전트는 사용하지 않는다 — 다른 플랫폼에서도 동작하도록 이식성을 우선했다.

```
사용자 요청
    ↓
CLAUDE.md (세션 시작 시 자동 로드) — 어떤 스킬을 쓸지 라우팅
    ↓
┌────────────┬─────────────┬────────────┐
│ wiki-ingest│  wiki-query │  wiki-lint │
│  (수집)    │   (질의응답)│   (점검)   │
└────────────┴─────────────┴────────────┘
    ↓
wiki/*.md 생성/갱신 (Obsidian이 그대로 열 수 있음)
```

각 스킬은 독립적으로 트리거된다. 사용자가 "X에 대해 알려줘" 하면 wiki-query가, "이 문서 위키에 넣어줘" 하면 wiki-ingest가 자동 실행된다.

---

## 설계 결정의 배경 — 실전에서 부딪힌 문제들

이 하네스는 처음부터 완성형으로 설계된 것이 아니라, 실제로 15개 소스 파일을 ingest하고 Obsidian으로 열어보는 과정에서 부딪힌 문제를 하나씩 해결해가며 만들어졌다. 각 설계 선택의 이유가 여기에 있다.

### 문제 1: 100개 파일을 한 번에 넣으면 컨텍스트가 터진다

**증상:** 소스가 많아지면 LLM이 모두 읽기 전에 컨텍스트 윈도우를 초과한다.

**해결:** 배치 처리 + 재개 가능한 진행 상태 저장.
- 기본 5개 파일/배치 (크기에 따라 3~10개 조절)
- 각 배치 후 `wiki/.ingest-progress.json`에 상태 저장
- 세션이 끊겨도 다음 실행 시 큐의 남은 파일부터 이어서 처리

→ `.claude/skills/wiki-ingest/skill.md`의 배치 처리 워크플로우

### 문제 2: 서머리가 점점 비대해져 소스의 복제본이 된다

**증상:** 다른 프로젝트에서 발견된 패턴 — ingest를 여러 번 하다 보면 summary 파일이 원본의 80%를 그대로 옮기게 되어 수백 줄로 불어난다. 서머리끼리 중복·모순이 생기고, query 시 LLM이 서머리만 읽고 정작 concept 페이지를 놓치는 문제 발생.

**해결:** Summary = **허브 페이지** 원칙.
- 소스 1개 = summary 1개
- **30줄 이내 엄수**
- 서머리에 넣는 것: 핵심 주장 1~3문장, 발견 3~7개 한 줄 bullet, 추출된 페이지 링크, 신뢰도 한 줄 평가
- 서머리에 넣지 않는 것: 데이터 테이블, 인용문, 상세 비교 — 이건 concept/entity 페이지의 역할
- 30줄 초과 시 상세 내용을 concept으로 이동, 서머리에는 링크만

→ `SCHEMA.md`의 Summary 페이지 규칙

### 문제 3: LLM이 지은 파일명이 기존 페이지와 겹친다

**증상:** `quantum-computing.md`가 이미 있는데 LLM이 다른 소스에서 같은 이름을 생성.

**해결:** 2단계 충돌 감지.
1. 후보 파일명 생성 → Glob으로 존재 여부 확인
2. 있으면 기존 내용과 비교 → 같은 주제면 갱신 모드, 다른 주제면 `-2`, `-3` 접미사

### 문제 4: 소스 파일명이 위키 페이지 이름으로 무의미하다

**증상:** `01-web-research.md`를 그대로 summary 이름으로 쓰면 Obsidian 그래프에서 노드가 의미 없어 보인다.

**해결:** **`{주제}-{관점/유형}.md`** 네이밍 규칙.
- 소스 `01-web-research.md` → summary `digital-twin-roi-web-survey.md`
- 주제는 concept/entity 페이지 이름과 겹칠 수 있으므로, 관점/유형을 붙여 구분
- concept은 "무엇에 대한 지식", summary는 "누가/어떻게 조사한 결과"이므로 자연스럽게 역할 분리

### 문제 5: sources/ 파일을 `[[wiki-link]]`로 참조하니 Obsidian 그래프에 유령 노드가 생긴다

**증상:** 본문에 `[[sources/01-web-research.md]]`라고 쓰면 Obsidian이 vault 밖 파일을 "존재하지 않는 페이지"로 인식해 점선 유령 노드로 그래프에 표시한다.

**해결:** sources 참조는 **wiki-link 대신 코드 형식** 사용.
- ❌ `[[sources/01-web-research.md]]`
- ✅ `` `sources/01-web-research.md` ``

vault 경계를 존중하는 것이 핵심 — Obsidian은 `wiki/` 폴더를 vault로 보고, `sources/`는 그 밖이다.

### 문제 6: Obsidian이 파일을 멋대로 수정한다

**증상:** Ingest 후 Obsidian을 열면 일부 파일의 YAML frontmatter가 한 줄로 합쳐지고, `[`가 `\[`로 이스케이프되고, `## title:` 같은 잘못된 마크다운으로 변환된다.

**근본 원인:** Obsidian의 **Properties 코어 플러그인**이 frontmatter를 자체 파서로 리라이트. 특히 `.obsidian/` 디렉토리가 프로젝트 루트에 생성되어 Obsidian이 전체 프로젝트를 vault로 인식했을 때 `.claude/skills/` 파일까지 건드리는 사고 발생.

**해결 2단계:**
1. `.obsidian/`을 `wiki/` 폴더 안에만 두기 — vault 범위를 제한
2. `wiki/.obsidian/core-plugins.json`에서 `"properties": false` 설정

→ CLAUDE.md의 Obsidian 호환 섹션

### 문제 7: 오케스트레이터 스킬이 과잉 설계였다

**초기 설계:** `wiki-llm` 오케스트레이터 스킬이 라우팅 담당.

**문제:** 스킬 라우팅·프로젝트 구조 정의는 모든 세션에서 필요하므로 CLAUDE.md가 적합. 오케스트레이터 스킬은 별도 트리거 단계를 추가할 뿐 가치가 없었다. 게다가 ingest/query/lint 스킬과 트리거 키워드가 겹쳐 혼란 유발.

**해결:** 오케스트레이터 스킬 삭제. CLAUDE.md로 라우팅 통합.

### 문제 8: log.md가 Obsidian 그래프에 노이즈로 들어왔다

**증상:** `wiki/log.md`가 vault 안에 있으니 Obsidian이 ingest/query/lint 이력을 모두 위키 페이지로 취급. 그래프에 "2026-04-12 ingest" 같은 노드가 등장, 시간이 갈수록 누적되어 그래프가 지저분해지고 검색 결과에도 섞임.

**해결:** log는 **지식이 아니라 운영 메타데이터**. vault 밖 `.wiki-log.md` (프로젝트 루트)로 이동.
- dot-prefix로 Obsidian이 기본 숨김 처리
- wiki 페이지들 간 `[[links]]`는 그대로 유지됨 (log→wiki 방향은 vault 밖이어도 무방)
- `.git/` 같은 메타데이터 파일과 같은 취급

반면 `index.md`는 vault 안에 유지 — 지식 카탈로그이자 그래프 허브 역할이므로 Obsidian에 노출되어야 함.

### 문제 9: query 스킬이 설계 형식을 따르지 않았다

**증상:** 스킬이 "답변 끝에 참조 페이지 목록 붙여라"라고 했는데, 실제 실행 시 LLM이 이를 무시하고 본문에 인라인 `(참조: ...)`로 대체. log.md 기록도 빠뜨림.

**해결:** 명령형을 피하고 **Why** 설명.
- "이 형식은 Obsidian에서 백링크 추적과 쿼리 감사를 가능하게 하므로 생략하지 않는다"
- "답변 생성과 로그 기록은 하나의 연산이다 — 답변만 하고 로그를 빠뜨리면 연산이 미완료된 것이다"
- 역기록 여부를 **항상** 명시하도록 형식에 포함 (없을 때도 "없음 — 기존 정보의 종합이므로")

→ Claude는 강압적 지시("ALWAYS")보다 이유를 이해했을 때 더 잘 따른다는 원칙의 실증.

---

## 설치 및 설정

### 1. 저장소 복제

```bash
git clone https://github.com/bjw202/wiki-llm-harness.git
cd wiki-llm-harness
```

### 2. Obsidian 설정

Obsidian에서 **wiki/ 폴더를 vault로** 연다 (프로젝트 루트가 아님).

```
File → Open vault → Open folder as vault → wiki/ 선택
```

Properties 코어 플러그인 비활성화:

```
Settings → Core plugins → Properties 토글 OFF
```

→ Claude가 생성한 frontmatter를 Obsidian이 파괴하지 않게 함.

### 3. Claude Code 준비

이 저장소의 `.claude/skills/`를 Claude Code가 자동 인식한다. 별도 설정 불필요. 프로젝트 루트에서 Claude Code를 실행하면 CLAUDE.md가 세션 시작 시 자동 로드된다.

---

## 사용법

### Ingest — 새 자료 수집

1. 원본 파일을 `sources/` 폴더에 넣는다 (PDF, 마크다운, 텍스트 등)
2. Claude에게 요청:

```
sources 폴더의 파일들을 위키에 추가해줘
```

LLM이 자동으로:
- 5개씩 배치 처리
- 소스별 summary 생성 (`wiki/summaries/`)
- 엔티티/개념 페이지 생성 또는 갱신 (`wiki/entities/`, `wiki/concepts/`)
- 여러 소스에 걸친 통찰을 synthesis로 기록 (`wiki/syntheses/`)
- 교차 참조와 index 갱신
- 진행 상태 저장

대량 파일이면 배치마다 중간보고하고 계속 여부를 묻는다. 중간에 끊겨도 다음 실행 시 이어서 처리된다.

### Query — 질의응답

```
디지털 트윈과 온톨로지가 관련 있어?
```

```
위키에 무엇이 있어?
```

LLM이 자동으로:
- index.md로 관련 페이지 식별
- Grep으로 키워드 검색 병행
- 관련 페이지를 읽고 종합
- 출처 포함 답변 생성
- 필요 시 새 통찰을 syntheses로 역기록
- `.wiki-log.md`에 쿼리 기록

답변 형식:

```
{답변 본문}

**참조한 위키 페이지:**
- [[페이지1]] — ...
- [[페이지2]] — ...

**역기록:** 없음 — 기존 정보의 종합이므로 역기록하지 않음
```

### Lint — 건강 점검

```
위키 점검해줘
```

LLM이 자동으로:
- frontmatter 유효성 (필수 필드, summary vs entity 필드 차이)
- 파일명 컨벤션
- 깨진 링크, 고아 페이지
- summary 30줄 초과 여부
- `[[sources/]]` 유령 노드 탐지
- 페이지 간 모순
- index 정합성

보고서 + 안전한 자동 수정 (링크 제거, index 동기화 등).

---

## 프로젝트 구조

```
wiki-llm-harness/
├── README.md              ← 이 문서
├── CLAUDE.md              ← 세션 시작 시 자동 로드 (라우팅, 초기화)
├── SCHEMA.md              ← 위키 규칙 (페이지 유형, frontmatter, 네이밍)
├── .wiki-log.md           ← 연산 기록 (ingest/query/lint 이력) — vault 밖
├── .gitignore             ← 위키 콘텐츠 제외 (구조만 유지)
│
├── .claude/skills/
│   ├── wiki-ingest/skill.md   ← 소스 수집 (배치+재개+충돌방지)
│   ├── wiki-query/skill.md    ← 질의응답 (출처+역기록+로그)
│   └── wiki-lint/skill.md     ← 건강 점검 (구조+내용+정합성)
│
├── sources/               ← 원본 자료 (불변, 읽기 전용) — gitignored
│
└── wiki/                  ← Obsidian vault 루트
    ├── index.md           ← 전체 페이지 카탈로그 (LLM이 먼저 읽음)
    ├── summaries/         ← 소스별 요약 허브 (30줄 이내)
    ├── entities/          ← 인물, 조직, 제품
    ├── concepts/          ← 이론, 기술, 방법론
    ├── syntheses/         ← 교차 분석, 통합 정리
    └── assets/            ← 이미지, 첨부파일
```

실제 위키 콘텐츠(`sources/`, `wiki/*/`)는 `.gitignore`로 제외되어 있다. 개인 지식은 공유되지 않으며, 구조만 저장소에 유지된다.

---

## 라이선스 / 크레딧

- 철학: [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- 구현: Claude Code 스킬 시스템 기반

이 하네스를 다른 프로젝트에 적용하려면 저장소를 복제하고 `sources/`에 자료를 넣은 뒤 Claude에게 ingest를 요청하면 된다. `CLAUDE.md`, `SCHEMA.md`, 스킬은 그대로 재사용 가능하다.
