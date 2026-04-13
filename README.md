# wiki-llm-harness

**Claude Code + Obsidian으로 만드는 개인 지식 위키.** 논문·기사·메모를 폴더에 넣고 "위키에 추가해줘"라고 말하면, Claude가 읽고·요약하고·연결하고·카탈로그까지 정리한다. 질문하면 위키에서만 답한다.

Andrej Karpathy의 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 패턴 구현.

---

## 이런 분께 추천

- 매일 읽는 논문·기사를 **영속적인 지식 베이스**로 축적하고 싶은 개인 연구자
- 팀의 도메인 지식을 **AI가 유지보수하는** 위키로 관리하고 싶은 스타트업
- 수동으로 Obsidian 쓰는 것이 지겨운 사람 — 링크·요약·정리를 LLM에 위임
- RAG 시스템을 직접 구축하는 것이 무거워 보이는 사람

**적합 규모**: &lt; 100K 토큰 (\~150-200 페이지). 그 이상이면 RAG 하이브리드 고려.

---

## 30초 사용 예시

```bash
# 1. 저장소 복제
git clone https://github.com/bjw202/wiki-llm-harness.git
cd wiki-llm-harness

# 2. sources/ 폴더에 자료 넣기 (마크다운·텍스트)
cp ~/Downloads/research-notes.md sources/

# 3. Claude Code 실행 후 요청
claude
> sources 폴더의 파일을 위키에 추가해줘
```

Claude가 자동으로 읽고, `wiki/` 폴더에 요약·엔티티·개념 페이지를 만든다.

```bash
# 4. 위키에 질문
> 이 논문에서 핵심 발견이 뭐야?

# 5. 주기적 건강 점검
> 위키 점검해줘
```

---

## 5분 빠른 시작 가이드

### Step 1 — 저장소 준비

```bash
git clone https://github.com/bjw202/wiki-llm-harness.git
cd wiki-llm-harness
```

폴더 구조가 이렇게 되어 있다:

```
wiki-llm-harness/
├── sources/       ← 여기에 원본 자료 넣기
├── wiki/          ← Claude가 여기에 위키 생성 (Obsidian vault)
├── .claude/skills/  ← 3개 스킬 (ingest, query, lint)
├── CLAUDE.md      ← Claude가 세션 시작 시 자동 로드
└── SCHEMA.md      ← 위키 규칙
```

### Step 2 — Obsidian 설정 (필수 3가지)

> ⚠️ **이 설정을 건너뛰면** Obsidian이 Claude가 생성한 파일의 YAML frontmatter를 파괴하는 [알려진 버그](https://forum.obsidian.md/t/yaml-properties-api-processfrontmatter-removes-alters-string-quotes-comments-types-formatting/65851)가 발생합니다.

1. `wiki/` **폴더를 vault로 연다** (프로젝트 루트 ❌, `wiki/` 폴더 ✓)

   - Obsidian → `File` → `Open vault` → `Open folder as vault` → `wiki/` 선택

2. **Settings → Editor → Properties in document →** `Source` 로 변경

3. **Settings → Core plugins** 에서 다음 토글 OFF:

   - `Properties` 토글 OFF
   - `Bases` 토글 OFF

3가지 모두 필요합니다. 하나만 놓쳐도 frontmatter가 깨질 수 있습니다.

### Step 3 — AI 에이전트 실행

사용하는 도구에 따라 설정이 다릅니다.

#### Claude Code 사용자

프로젝트 루트에서 `claude` 명령으로 실행합니다. `CLAUDE.md`가 자동 로드되어 프로젝트 규칙을 이해합니다. 추가 설정 불필요.

#### Cline (VS Code 확장) 사용자

Cline은 `CLAUDE.md`를 자동 로드하지 않습니다. `.clinerules/` 폴더를 만들고 그 안에 CLAUDE.md를 넣어야 인식합니다.

**방법 A — 이동 (Cline만 사용할 때):**

```bash
mkdir -p .clinerules
mv CLAUDE.md .clinerules/CLAUDE.md
```

**방법 B — 심볼릭 링크 (Claude Code와 Cline 모두 사용할 때, 권장):**

```bash
mkdir -p .clinerules
ln -s ../CLAUDE.md .clinerules/CLAUDE.md
```

이렇게 하면 원본은 프로젝트 루트에 두고 Cline도 동일 파일을 참조합니다. 한쪽에서 수정해도 양쪽에 반영됩니다.

> Windows 사용자는 심볼릭 링크가 제약이 있을 수 있으니 방법 A를 권장합니다.

### Step 4 — 첫 자료 넣어보기

```bash
# 마크다운(.md), 텍스트(.txt) 지원
cp ~/some-article.md sources/
```

Claude에게:

```
sources 폴더의 파일을 위키에 추가해줘
```

Claude가 다음을 자동 수행:

- ✅ 파일 읽기 (여러 개면 5개씩 배치)
- ✅ 소스별 **요약 페이지** 생성 (`wiki/summaries/`)
- ✅ 등장한 **인물/조직/제품** → `wiki/entities/`
- ✅ 등장한 **개념/이론** → `wiki/concepts/`
- ✅ 여러 자료에 걸친 통찰 → `wiki/syntheses/`
- ✅ `wiki/index.md` 자동 갱신
- ✅ Obsidian에서 바로 열람 가능

중간에 멈춰도 다음에 이어서 처리됩니다.

### Step 5 — 위키에 질문

```
양자 컴퓨팅에 대해 알려줘
```

```
이 논문과 저 논문의 공통점이 뭐야?
```

Claude는 **위키에 있는 정보만으로** 답변하며, 어느 페이지에서 가져왔는지 출처를 함께 보여줍니다. 위키에 없는 주제는 외부 지식으로 채우지 않고 "없다"고 정직하게 답합니다.

---

## 3대 기능 사용법

### 📥 Ingest — 새 자료 수집

**언제 쓰나**: `sources/` 폴더에 파일을 추가한 뒤

**어떻게**: 아래 중 아무렇게나 말해도 됩니다.

- "sources 폴더의 파일 전부 위키에 추가해줘"
- "이 문서를 위키에 넣어줘"
- "새로 추가한 자료 정리해줘"

**결과**: `wiki/` 폴더에 4가지 유형의 페이지가 생성됩니다.

| 유형 | 역할 | 예시 |
| --- | --- | --- |
| Summary | 소스 1개당 허브 페이지 (30줄 이내) | `digital-twin-web-survey.md` |
| Entity | 고유명사 (인물·회사·제품) | `andrej-karpathy.md`, `siemens.md` |
| Concept | 추상 개념 (이론·기법) | `wiki-llm.md`, `transformer.md` |
| Synthesis | 교차 분석 | `rag-vs-wiki-comparison.md` |

### 🔍 Query — 질의응답

**언제 쓰나**: 위키에 쌓인 지식이 궁금할 때

**어떻게**:

- "X에 대해 알려줘"
- "X와 Y의 관계는?"
- "위키에 뭐가 있어?"
- "\~사례 있어?"

**답변 형식**:

```
{답변 본문}

참조한 위키 페이지:
- [[페이지1]] — 이 페이지에서 가져온 정보
- [[페이지2]] — ...

역기록: 없음 — 기존 정보의 종합이므로 역기록하지 않음
```

**역기록(write-back)**: 여러 페이지를 조합해 **새로운 통찰**이 발견되면 Claude가 자동으로 `syntheses/`에 새 페이지를 만들어 저장합니다. 단순 조회는 역기록하지 않습니다.

### 🧹 Lint — 건강 점검

**언제 쓰나**: 주기적으로 (주 1회 정도 권장)

**어떻게**:

- "위키 점검해줘"
- "위키 상태 봐줘"

**무엇을 점검하나**:

- frontmatter 필수 필드 누락
- 깨진 `[[링크]]`
- 고아 페이지 (어디서도 참조 안 됨)
- Summary 30줄 초과
- 페이지 간 모순
- Index.md 정합성
- `needs-review: true` 플래그 (분류 애매한 페이지)

안전한 수정(링크 정리, index 동기화)은 자동 적용. 모순·중복은 보고만 하고 사용자 판단에 맡깁니다.

---

## 자주 묻는 질문 (FAQ)

### Q. Cline·Cursor 등 다른 AI 에이전트에서도 쓸 수 있나요?

스킬(`.claude/skills/`)은 구조만 있으면 어느 에이전트에서든 읽을 수 있어 호환됩니다. 차이는 **루트 규칙 파일 위치**:

| 도구 | 규칙 파일 경로 |
|------|-------------|
| Claude Code | `CLAUDE.md` (프로젝트 루트) |
| Cline | `.clinerules/CLAUDE.md` |
| Cursor | `.cursorrules` 또는 `.cursor/rules/` |

[Step 3](#step-3--ai-에이전트-실행)의 심볼릭 링크 방식을 쓰면 여러 도구 동시 지원 가능합니다.

### Q. Obsidian 없이도 쓸 수 있나요?

네. 위키는 그냥 마크다운 파일이라 VS Code나 어떤 편집기에서든 열 수 있습니다. Obsidian은 **그래프 뷰·백링크·검색**이 편해서 추천하는 것뿐입니다.

### Q. 위키에 쌓인 정보가 공개 저장소에 올라가지 않나요?

안 올라갑니다. `.gitignore`에서 `sources/`와 `wiki/*/`를 모두 제외합니다. 저장소에는 **구조와 스킬만** 올라가고, 개인 지식은 로컬에만 남습니다.

### Q. 어떤 파일 형식을 지원하나요?

현재는 **텍스트 기반 파일**만 지원합니다 — 마크다운(`.md`), 일반 텍스트(`.txt`). PDF·Word·HTML 등은 미리 마크다운·텍스트로 변환해서 `sources/`에 넣어야 합니다. PDF 직접 파싱은 향후 지원 예정.

### Q. Claude가 frontmatter를 잘못 만들면요?

`위키 점검해줘` 실행하면 lint가 필수 필드 누락을 자동 수정합니다. 반복되면 스킬 파일을 조정해야 할 수 있습니다.

### Q. 위키가 커지면 느려지지 않나요?

\~100-200 페이지까지는 쾌적합니다. 그 이상이면 Query 시 컨텍스트 부담이 생기므로 RAG 하이브리드를 고려하세요. wiki-llm은 **"압축된 핵심 지식"**, RAG는 \*\*"전체 자료"\*\*로 역할 분담이 자연스럽습니다.

### Q. 다른 프로젝트에 같은 하네스를 쓰고 싶어요

이 저장소를 복제한 뒤 `sources/`와 `wiki/`만 비우면 됩니다. `CLAUDE.md`, `SCHEMA.md`, `.claude/skills/` 는 그대로 재사용 가능합니다.

### Q. 언어는 한국어만 되나요?

아니요. 스킬은 한국어로 작성되어 있지만 위키 페이지 언어는 **소스 언어를 따릅니다**. 영어 논문을 넣으면 영어 위키가 생깁니다. 필요하면 "한국어로 작성해줘"라고 요청 가능.

---

## 문제 해결 (Troubleshooting)

### 🐛 frontmatter가 한 줄로 합쳐졌어요 (`## title: "..." tags:...`)

**원인**: Obsidian의 `processFrontMatter` 버그. 위의 [Step 2 Obsidian 설정](#step-2--obsidian-%EC%84%A4%EC%A0%95-%ED%95%84%EC%88%98-3%EA%B0%80%EC%A7%80) 3가지 모두 적용했는지 확인하세요.

**수정**: Obsidian을 **닫은 상태에서** Claude에게 "위키 점검해줘"라고 하면 자동 수정됩니다.

### 🐛 Obsidian 그래프에 `01-web-research` 같은 유령 노드가 보여요

**원인**: 위키 페이지 안에서 `[[sources/파일명]]` 형태로 참조했기 때문. `sources/`는 vault 밖이라 Obsidian이 "존재하지 않는 페이지"로 인식합니다.

**수정**: "위키 점검해줘" — Lint가 `[[sources/...]]`를 `` `sources/...` `` 코드 형식으로 자동 변환합니다.

### 🐛 Ingest 중간에 멈췄어요

**원인**: 컨텍스트 윈도우 초과 또는 세션 종료.

**수정**: 다시 "sources 폴더 이어서 처리해줘"라고 하면 `wiki/.ingest-progress.json`을 보고 큐의 남은 파일부터 처리합니다.

### 🐛 같은 주제가 여러 파일로 흩어져 있어요

**원인**: LLM이 네이밍 충돌을 감지 못한 경우 (드물지만 발생).

**수정**: "위키 점검해줘" → Lint가 중복 탐지하여 보고. 사용자가 병합 여부 결정.

### 🐛 위키에 있을 법한 정보인데 "없다"고 해요

**원인**: query 스킬이 검색에 실패한 경우. 인덱스 제목에 키워드가 없고 Grep도 동의어를 못 잡으면 발생.

**해결**: 더 구체적인 질문으로 다시 물어보거나, `wiki/index.md`를 직접 열어 관련 페이지를 확인한 뒤 페이지 이름을 명시하여 재질문.

---

## 프로젝트 구조

```
wiki-llm-harness/
├── README.md              ← 이 문서
├── CLAUDE.md              ← Claude 세션 초기 로드 (라우팅)
├── SCHEMA.md              ← 위키 규칙 (페이지 유형, frontmatter)
├── .wiki-log.md           ← 연산 이력 (ingest/query/lint)
│
├── .claude/skills/
│   ├── wiki-ingest/       ← 소스 수집 스킬
│   ├── wiki-query/        ← 질의응답 스킬
│   └── wiki-lint/         ← 건강 점검 스킬
│
├── sources/               ← 원본 자료 (gitignore)
│
└── wiki/                  ← Obsidian vault (gitignore)
    ├── index.md           ← 페이지 카탈로그
    ├── summaries/         ← 소스별 요약 허브 (30줄 이내)
    ├── entities/          ← 고유명사
    ├── concepts/          ← 추상 개념
    ├── syntheses/         ← 교차 분석
    └── assets/            ← 이미지·첨부
```

---

## 철학과 설계 배경 (더 알고 싶으면)

이 하네스는 처음부터 완성형으로 설계되지 않았습니다. 실제 자료를 넣고 Obsidian으로 열어보는 과정에서 부딪힌 문제를 하나씩 해결하며 만들어졌습니다. 각 설계 선택의 이유가 아래에 있습니다.

&lt;details&gt; &lt;summary&gt;&lt;b&gt;Karpathy의 철학 — "장부 정리를 LLM이 전담한다"&lt;/b&gt;&lt;/summary&gt;

Karpathy의 gist를 관통하는 한 문장:

> "지식 기반 유지보수의 지루한 부분은 읽기나 사고가 아니라 \*\*장부 정리(bookkeeping)\*\*다."

기존 RAG는 질의마다 원본을 재발견합니다. 매번 인터프리터가 돌아가는 셈. wiki-llm은 다르게 접근합니다 — 소스를 **한 번 컴파일**해서 영속적 마크다운 위키로 저장하고, 질의 시에는 이 위키만 읽습니다. 컴파일러와 인터프리터의 차이입니다.

**3계층 아키텍처:**

| 계층 | 역할 | 소유자 |
| --- | --- | --- |
| Raw Sources | 불변 원본 자료 | 인간이 큐레이션 |
| Wiki | LLM이 유지하는 마크다운 | LLM이 관리 |
| Schema | 위키 규칙·컨벤션 | 인간이 설계, LLM이 준수 |

**3대 오퍼레이션**: Ingest / Query / Lint

**4가지 페이지 유형**: Summary / Entity / Concept / Synthesis

&lt;/details&gt;

&lt;details&gt; &lt;summary&gt;&lt;b&gt;실전에서 부딪힌 9가지 문제와 해결&lt;/b&gt;&lt;/summary&gt;

**문제 1: 100개 파일 ingest 시 컨텍스트 터짐**→ 배치 처리(5개/배치) + `.ingest-progress.json`으로 재개 가능

**문제 2: Summary가 소스 복제본이 됨**→ Summary = 30줄 이내 허브 페이지 원칙. 상세는 concept/entity로

**문제 3: LLM이 기존 파일명과 충돌하는 이름 생성**→ 2단계 충돌 감지 (Glob → 내용 비교 → 갱신 or 접미사)

**문제 4: 소스 파일명이 위키에서 무의미**→ `{주제}-{관점}.md` 네이밍 (`01-web-research` → `digital-twin-web-survey`)

**문제 5:** `[[sources/xxx]]` **링크가 Obsidian 유령 노드 생성**→ vault 밖 파일은 `` `sources/xxx` `` 코드 형식으로 참조

**문제 6: Obsidian이 파일을 멋대로 수정 (frontmatter 파괴)**→ 3중 방어: vault 범위 제한 + `propertiesInDocument: source` + Properties/Bases 비활성화

**문제 7: 오케스트레이터 스킬이 과잉 설계**→ 삭제. 라우팅은 CLAUDE.md로 통합

**문제 8: log.md가 Obsidian 그래프 노이즈**→ vault 밖 `.wiki-log.md`로 이동. 운영 메타데이터는 지식과 분리

**문제 9: query 스킬이 설계 형식을 무시**→ 명령형 대신 Why 설명. "이 형식은 Obsidian 백링크 추적을 가능하게 한다"

&lt;/details&gt;

&lt;details&gt; &lt;summary&gt;&lt;b&gt;다른 프로젝트에 적용하는 법&lt;/b&gt;&lt;/summary&gt;

이 저장소를 템플릿으로 복제:

```bash
git clone https://github.com/bjw202/wiki-llm-harness.git my-knowledge-base
cd my-knowledge-base
rm -rf .git
git init
```

재사용 가능한 것:

- `CLAUDE.md` — 도메인에 맞게 설명 부분만 수정
- `SCHEMA.md` — 대부분 그대로 사용 가능
- `.claude/skills/` — 모두 그대로 재사용

도메인별 조정이 필요한 것:

- Entity vs Concept 경계 사례 (도메인 특유 용어가 있으면 `wiki-ingest/skill.md`의 경계 사례 표에 추가)

&lt;/details&gt;

---

## 크레딧

- 철학: [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- 구현: Claude Code 스킬 시스템 기반
- Repo: https://github.com/bjw202/wiki-llm-harness