# Wiki-LLM

Karpathy의 LLM Wiki 철학에 기반한 지식 위키 프로젝트. 인간이 소스를 큐레이션하고 방향을 지시하면, LLM이 장부 정리(bookkeeping)를 전담한다.

## 프로젝트 구조

위키 규칙(페이지 유형, frontmatter, 네이밍 컨벤션 등)은 `SCHEMA.md`에 정의되어 있다. 위키 작업 전 `SCHEMA.md`를 먼저 읽는다.

## 3대 연산과 스킬

| 연산 | 스킬 | 용도 |
|------|------|------|
| **Ingest** | `wiki-ingest` | 소스 수집 → summary + 개념/엔티티 페이지 생성. 배치 처리, 재개 가능 |
| **Query** | `wiki-query` | 위키 검색 → 출처 기반 답변. 역기록 판단 포함. log.md 기록 필수 |
| **Lint** | `wiki-lint` | 위키 건강 점검 → 모순·고아·깨진 링크 탐지·수정 |

에이전트 없이 스킬만으로 동작한다. 다른 플랫폼 이식성을 위한 의도적 결정.

## 초기화

위키 작업 전 아래 파일/디렉토리가 존재하는지 확인한다. 없으면 생성한다:
- `SCHEMA.md`, `wiki/index.md`, `wiki/log.md`
- `sources/`, `wiki/summaries/`, `wiki/entities/`, `wiki/concepts/`, `wiki/syntheses/`

## Obsidian 호환

- vault 경로: `wiki/` 폴더 (프로젝트 루트가 아님)
- `sources/` 파일은 `[[wiki-link]]`로 참조하지 않는다 — vault 밖이므로 `` `sources/파일명` `` 코드 형식 사용
- Properties 코어 플러그인은 비활성화 권장 (frontmatter 파손 방지)
