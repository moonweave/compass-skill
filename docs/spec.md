# `/compass` Skill Design (v2 — pivoted to drift + rot)

- **Date:** 2026-05-03
- **Status:** Approved (brainstorm complete, ready for implementation plan)
- **Author:** Moon-young / Claude Opus 4.7
- **Origin session:** continuation of `/decide` skill build (project_decide_skill.md)
- **Pivot history:** v1 draft was duplicate + architecture (axes 1+2). Advisor review flagged ~70-80% overlap with `/decide`. v2 pivots to **drift + rot** (axes 3+4) — the parts where nothing else covers and the user's most viscerally stated pain ("코드베이스가 돌아서니 망가져 있거나" + "Claude가 잘못된 길로 누적 표류").

---

## 1. Problem & Motivation

LLM 코딩 에이전트(Claude Code, Codex, Cursor 등) 사용자가 한 작업을 long-session으로 진행할 때 두 가지 누적 위험:

1. **LLM Drift** — 사용자 첫 의도에서 점점 멀어지는 변경이 쌓임. LLM은 "사용자 최근 요청대로 했다"는 정렬은 유지하지만 처음 의도와의 정합성은 검증하지 않음. arXiv 2509.18970 ("LLM-based Agents Suffer from Hallucinations")이 long-horizon hallucination 누적 mitigation으로 단계별 self-assessment 권장.

2. **Code/Architecture Rot** — 변경이 누적되면서 코드베이스가 망가짐. uncommitted 폭증, test/lint 깨짐, 파일 비대화, 모듈 경계 깨짐 (저수준 디테일을 고수준이 직접 import), circular dep, naming/디렉터리 패턴 위반 등.

기존 도구는 두 빈 칸을 채우지 못함:
- **시작 시점**: `superpowers:brainstorming` (의도 탐색만)
- **완성 시점**: `/review`, `code-reviewer`, `pr-review-toolkit`, `verification-loop` (PR 게이트, 사후)
- **내부 friction**: `claude-coach-plugin` (사용자 짜증 시그널 → CLAUDE.md 갱신, drift 판정은 안 함)
- **plan vs code 갭**: `drift-analysis` v5.1 (좁은 plan drift만, 우리 정의의 LLM drift 아님)
- **결정 도우미**: `/decide` (단일 결정 시점, 누적 표류는 다루지 않음)

**`/compass`는 "long-session 표류 + 코드베이스 부패를 internal data로 점검"하는 빈 자리**를 채운다. 사용자 철학("Claude가 잘못된 길 따라가는 거 막는다")을 코드화.

## 2. Concept

`/compass`는 **개발 도중 long-session 표류와 코드베이스 부패를 점검**하는 슬래시 스킬. 두 축:

1. **Drift Check** — 현재 작업이 처음 의도에서 표류했는가?
   - 데이터: 현재 transcript의 첫 user 메시지 (또는 명시 인자) + 최근 N턴 user/assistant 메시지
   - 비교: 첫 의도 vs 최근 작업 방향, 정합성 판정

2. **Rot Check** — 코드베이스가 누적 변경으로 망가지고 있는가?
   - 데이터: git diff/status, test/lint 결과 (smart cache), 파일 size, TODO grep, import graph, 디렉터리 구조
   - 판정: file-level signals (mechanical) + architecture-level (circular dep + LLM boundary judgment)

`/decide`와 차별: external expert 비교가 1급 시민이 아니라 **internal session·codebase 데이터**가 1급 시민. 외부 reference는 보조 (필요 시 보충 정도).

## 3. Design Decisions

| Q | Decision | Rationale |
|---|---|---|
| Pivot | drift + rot (axes 3+4) | advisor 지적: 1+2는 `/decide`와 ~70% 겹침. 3+4가 사용자 진짜 통증 + 빈 칸 |
| P1 Drift baseline | C — 명시 인자 우선 + auto-extract fallback (최근 spec → task-init 휴리스틱 → 사용자 명시) | `/decide` UX 일관, spec 있을 때 정확도 ↑, **첫 user msg 폐기** (long-session multi-task 일반 사용 패턴 반영) |
| P2 Rot signals | C+ — file-level {1,2,3,4,5} + architecture {9,10} | 사용자 통증 두 축(test/파일비대 + 모듈경계) 모두 커버 |
| P2 Test 실행 | iii — smart cache (10분 내 결과 있으면 사용, 없으면 stale 표시) | cost ↓ + 자연 활용 |
| P3 트리거 | A — manual slash only | hook cascade trauma, /decide 패턴 일관, dogfooding 우선 |
| P4 출력 | grid + per-axis evidence + verdict | drift-analysis 검증 패턴 |
| Action chain | drift→brainstorming, rot→simplify/investigate | advisor concern 해소 (/decide 일색 아님) |

## 4. Inputs & Triggers

**호출 형태:**
- `/compass` — 인자 없음. drift baseline auto-extract.
- `/compass <intent>` — 명시 baseline. 인자 있으면 우선.

**Drift baseline auto-extract fallback chain (인자 없을 때):**

> ⚠️ 첫 user 메시지를 단순히 baseline으로 쓰면 안 됨 — long session은 다수 task를 다루고 (예: 본 spec 작성 세션도 `/decide` → `/compass` 두 task), 첫 메시지가 현재 task와 무관할 가능성 높음. compaction까지 끼면 더 심해짐.

1. **최근 spec (primary)** — `~/docs/superpowers/specs/`의 최근 24h 내 mtime spec 있으면 그것이 baseline. spec 내용에서 §1 또는 §2를 의도 추출 source로.
2. **Task-init 휴리스틱** — transcript jsonl을 **역순 스캔**, 가장 최근의 task-init 시그널 (skill 호출: `/decide`, `/compass`, `superpowers:brainstorming`, `/team`, `/research-team` 등) 직후 user 메시지를 baseline으로. 시그널 없으면 가장 최근 새 토픽 진입 user 메시지 (휴리스틱: 직전 assistant 응답이 결론/요약 형태이고 user가 새 명사 도입).
3. **사용자 명시 요청** — 위 둘 다 실패 시 "이번 task 의도가 뭐야? 한 줄로 알려줘" 일회성 turn.

첫 user 메시지를 무조건 잡지 않음 — 명시적 폐기 (advisor 검증).

**자동 트리거:** 없음 (P3).

**환경 detection (rot signal 도구 선택용):**
| Marker (cwd) | rot 도구 |
|---|---|
| `package.json` | `npm test` (cache), `eslint`, madge (circular dep) |
| `pyproject.toml` / `requirements.txt` | `pytest` (cache), `ruff`, pydeps |
| `Cargo.toml` | `cargo test`, `clippy` |
| `go.mod` | `go test`, `go vet` |
| 마커 없음 | git + 파일 size + TODO grep만 (architecture LLM judgment은 항상 가능) |

## 5. Two-Axis Pipeline

```
/compass [<intent>]
        │
        ├──→ Axis 1: Drift Check (sequential, transcript 단일 source)
        │       ├─ baseline 결정 (§4 fallback chain: 인자 → 최근 spec → task-init 휴리스틱 → 사용자 명시)
        │       ├─ 최근 N=10 user+assistant 메시지 추출 (transcript jsonl 마지막에서 역순)
        │       └─ Opus judgment: baseline ↔ recent 정합성, drift 신호 추출
        │
        ├──→ Axis 2: Rot Check (parallel data collection)
        │       ├─ git status / git diff --stat (mechanical)
        │       ├─ Test result cache (artifact mtime 기반): `.pytest_cache/lastfailed`, `coverage/*`, `target/test-results/`, `node_modules/.cache/jest` 등 표준 artifact 확인 → 10분 내 mtime 있으면 read+사용, 10분 초과면 stale 표시, artifact 미가용이면 'no cache' skip. compass가 직접 test 실행 안 함 (cost 회피 + 사용자 직접 돌린 흔적 활용 명분 보존)
        │       ├─ Lint 실행 (cheap, 수 초): ruff/eslint/clippy
        │       ├─ 파일 size: 최근 N=14일 변경 파일 중 line count >500
        │       ├─ TODO/FIXME/XXX grep count
        │       ├─ Circular dep: madge/pydeps (도구 가용 시만, 실패 graceful)
        │       └─ Opus judgment: 최근 변경 파일 import 헤더 + 디렉터리 트리 → boundary degradation 판정
        │
        └──→ Synthesizer (Opus, single call)
                ├─ 두 축 결과 통합
                ├─ severity 판정 per axis (SAFE/SUSPICIOUS/CRITICAL)
                ├─ overall verdict (PROCEED/CONSIDER/STOP)
                └─ Top action
```

**Trust Boundary:** internal data가 본질이라 외부 콘텐츠 의존 ↓. 그러나 transcript 안에는 사용자가 fetch한 외부 콘텐츠가 섞여있을 수 있음 (이전 turn에서 web 페이지 인용 등) → **transcript 텍스트 자체도 untrusted instruction으로 취급 안 함**. baseline 추출은 의미 매칭만, instruction parsing 절대 안 함 (`/decide` C3 패턴).

**Rot signal 임계값 (mechanical signals 정량):**

| Signal | SAFE | SUSPICIOUS | CRITICAL |
|---|---|---|---|
| Git uncommitted | ≤4 files AND ≤200 lines | 5-9 files OR 200-500 lines | ≥10 files OR ≥500 lines |
| Test (cache 가용 시) | all pass | 1-3 fail | ≥4 fail or compile error |
| Test (stale, 10분+) | "stale" 표시 — severity 판정 보류 | — | — |
| Lint | ≤5 warnings, 0 errors | 6-20 warnings OR 1-3 errors | ≥21 warnings OR ≥4 errors |
| File bloat | 0 files >500 lines (recently modified) | 1-2 files >500 | ≥3 files >500 OR 1 file >1000 |
| TODO grep | ≤10 | 11-30 | ≥31 |
| Circular dep | 0 | 1-2 | ≥3 |
| Architecture (LLM) | 경계 명확, 패턴 일관 | minor 위반 1-2건 | major 위반 (계층 breach, 다수 무관 import, 신규 파일 패턴 무시) |

**Rot 축 종합:** 위 신호들 중 CRITICAL 1개 이상이면 axis = CRITICAL. **SUSPICIOUS 1개 이상**이면 axis = SUSPICIOUS. 그 외 SAFE. (단일 의심 신호도 SUSPICIOUS로 노출 — advisor 검증.)

**Drift signal 임계값 (semantic, Opus judgment):**

| | SAFE | SUSPICIOUS | CRITICAL |
|---|---|---|---|
| Drift | 최근 N턴 작업이 baseline과 직접 align | minor pivot (의도 ⊃ 현재 작업, scope 좁아짐 등) | 정반대 방향 (예: baseline은 "X 만들기"인데 최근 작업은 "X 폐기 후 Y") OR scope creep (의도에 없던 새 시스템 누적) |

**Verdict 합산:**
- 두 축 SAFE → **PROCEED**
- 한 축 SUSPICIOUS, 다른 축 SAFE → **CONSIDER**
- 한 축 CRITICAL OR 두 축 모두 SUSPICIOUS → **STOP**

**실패 모드:**
- Transcript 접근 실패 (권한/경로) → drift axis = "INSUFFICIENT", rot만 출력
- 모든 rot 도구 실패 → rot axis = "INSUFFICIENT", drift만 출력
- 양쪽 다 실패 → "cannot collect signals, check tool availability" 후 종료

## 6. Output Template

```
## /compass — <session intent 한 줄 요약>

### Drift Check  [SAFE | SUSPICIOUS | CRITICAL]
<한 줄 요약>

- Initial intent: "<첫 user msg 또는 명시 인자 한 줄 요약>"
- Recent N turns: <최근 작업 한 줄 요약>
- Drift signal: <구체적 갭 1-2줄, transcript line 참조 가능>

### Rot Check  [SAFE | SUSPICIOUS | CRITICAL]
<한 줄 요약>

- Git: <X files / Y lines uncommitted>
- Tests: <pass/fail counts, or "stale (cache age: Nm)">
- Lint: <W warnings / E errors>
- File bloat: <files >500 lines, recently modified>
- TODO: <count>
- Architecture: <circular deps + LLM boundary judgment 1-2줄>

---

**Verdict: PROCEED | CONSIDER | STOP**

<verdict 한 줄 근거>

**Top action:**
<후속 행동 — table에서 매핑>

**Considered:** <기각된 후보 1-2개 + 기각 이유, 해당 시>
```

**후속 행동 chain (advisor concern 해소 — `/decide` 의존도 ↓):**

| 축 | severity | Top action |
|---|---|---|
| Drift | SUSPICIOUS | "의도 재명시: `/compass <intent>`로 baseline 갱신, 또는 명시적 pivot 선언" |
| Drift | CRITICAL | "STOP — 처음 의도와 정반대 방향. 새 세션 권장 또는 `superpowers:brainstorming`으로 새 의도 재정의" |
| Rot | SUSPICIOUS | "cleanup commit + lint fix 권장" |
| Rot | CRITICAL | "STOP — refactor 필요. `simplify` 스킬 / 휴식 후 작은 단위로 작업 분할 / `investigate`로 깨진 부분 root-cause" |
| 두 축 모두 CRITICAL | — | "STOP — 새 세션 강력 권장. 현재 세션 commit/stash 후 fresh로 재시작" |

**Considered 라인 의무:** rot signal 중 CRITICAL 안 잡힌 것이 있으면 명시 ("test stale로 보류 / circular dep 0 / etc"). 검색 범위 투명성 (`/decide` 검증 패턴).

**언어:** 사용자 응답 매칭 (한국어 입력 → 한국어, 영어 → 영어).

## 7. Out of Scope (v1)

명시적 YAGNI — v1.5/v2 후보:

- **Duplicate / Architecture standard 비교 (원래 v1 axes 1+2)** — `/decide` 와 ~70% 겹침 확인되어 pivot. 필요 시 `/decide` 호출로 보강.
- **Stop hook / N턴 자동 트리거** — P3 결정. cascade trauma 회피, dogfooding 후 v1.5 D nudging 검토.
- **Test 자동 실행 (smart cache 외)** — cost 통제 위해 사용자가 직접 돌린 결과만 활용. `/compass --with-test`는 v1.5 옵션.
- **결과 영속화 (sqlite trend tracking)** — claude-coach 패턴, 단일 사용자엔 오버킬.
- **Cross-session drift** — 현재 세션 transcript만. 여러 세션 누적 drift는 v2.
- **Codex/Cursor transcript 호환** — Claude Code transcript jsonl 한정. 다른 에이전트는 v2.
- **자체 테스트 자동화** — fresh 세션 cold test로 검증.

## 8. Success Criteria

1. **Functional**: `/compass [<intent>]` 호출 시 §5 pipeline 실행 + §6 형식 출력 + verdict 분기 동작
2. **Trust boundary**: transcript 안의 어떤 텍스트도 instruction으로 해석 안 됨 (의도 추출 vs instruction parsing 분리 테스트 1건)
3. **Drift baseline auto-extract**: 인자 없이 호출 시 첫 user 메시지 정확 추출 (3개 케이스 — 짧은 메시지 / 긴 멀티주제 / 코드 블록 포함)
4. **Rot signal graceful**: 도구 1-2개 실패 시 axis severity 판정 가능 (남은 signal로). 모든 도구 실패 시 INSUFFICIENT 명시.
5. **Test cache 정확**: 10분 내 result 있으면 사용 + age 표시, 없으면 skip + stale 표시
6. **Cold-read test**: fresh 세션에서 description만 보고 사용 의도 파악 가능
7. **Real-world dogfooding**: 사용자 본인 다음 long-session에서 `/compass` 호출 → 실제 drift/rot 잡혔거나 SAFE 판정 신뢰됨

## 9. Technical Constraints

- **모델**: opus (semantic judgment 품질, `/decide` 동일)
- **도구**: Bash (transcript jsonl read, git, test, lint, grep), Read (transcript), mcp tool 거의 안 씀 (외부 의존 ↓)
- **형식**: Claude Code skill (SKILL.md + YAML frontmatter)
- **Trigger gating**: description-gated 자동 발동 차단, literal `/compass` 슬래시만
- **언어**: 사용자 응답 매칭
- **Transcript 경로 가정**: `~/.claude/projects/<project-id>/<session-id>.jsonl` (Claude Code 표준), 다른 경로면 fallback chain step 3 진입

## 10. Repo & Sync Structure

`/decide` 표준 절차 그대로 차용 (project_decide_skill.md 메모리에 문서화):

- **실제 개발**: `~/ai-skills-dev/my-skills/compass-skill/`
- **Claude Code 로드**: `~/.claude/skills/compass → ../../ai-skills-dev/my-skills/compass-skill` (relative symlink)
- **GitHub backup**: `https://github.com/Moon-python/compass-skill` (public, MIT)
- **워크플로우**: 편집 → fresh 세션 자동 반영 → git commit → push

## 11. Inspiration Sources Studied

리서치 단계(2026-05-03)에서 raw SKILL.md / agent definition 직접 읽음:

| Source | 학습한 패턴 | `/compass`(v2)에 반영 |
|---|---|---|
| ECC `search-first` | (rejected) — v1 draft (axes 1+2) 영감이었으나 pivot 후 v2 axes 3+4와 직접 관련 없음 | — |
| ECC `architect` | Read-only 도구 제한, Red Flags 명명, opus 모델 | Rot 축 LLM boundary judgment 영감, opus |
| ECC `verification-loop` | Continuous Mode 개념, PASS/FAIL grid | grid 출력, Continuous는 v2 |
| `drift-analysis` v5.1 | 이중 레이어 (collector → synthesizer), 3-tier confidence + 구체 임계값, priority calculation | Rot signal 정량 임계값, synthesizer 단일 Opus, 명명("drift") 차용 |
| `claude-coach-plugin` | Stop hook transcript 분석, sqlite event store, Haiku 비용 | Drift 축 transcript 분석 영감 (자동 hook은 v2) |
| **`/decide` (본인 직전 스킬)** | manual-only, auto-extract fallback, trust boundary C3, opus, output budget, "Considered:" 라인, sync 구조 | UX 패턴 그대로 차용. 본 스킬과 결정 시점 분리 (compass = 누적 점검, decide = 단일 결정) |

추가 학계 reference: arXiv 2509.18970 ("LLM-based Agents Suffer from Hallucinations") — long-horizon hallucination 누적의 mitigation으로 단계별 self-assessment 권장. `/compass`는 그 self-assessment를 internal data 비교로 구현.

## 12. Open Questions for v2+

- **drift baseline 진화**: long session에서 의도가 정당하게 진화하는 경우(scope 명시 변경) vs 누적 표류 구분이 LLM judgment 만으로 충분한가? 사용자 명시 pivot 선언 메커니즘 필요할까?
- **Rot 트렌드**: 단발 점검은 절대값. trend(시간에 따른 변화) 추적이 더 의미 있을 수 있음 — 그러려면 영속화 필요
- **Codex/Cursor transcript 호환**: 사용자가 다른 에이전트도 쓰면 `/compass`가 그쪽도 봐야 하는데 transcript 형식 다름. adapter 구조 검토 필요
- **N=10 turns 적정값**: 직전 N턴 추출의 N이 작으면 noise, 크면 dilution. dogfooding 후 조정
- **Test cache TTL 10분 적정값**: 사용자 작업 속도에 따라 다름 (빠른 iteration vs 느린 작업). config화 필요할 수도

이 질문들은 v1 dogfooding 후 사용 패턴 보고 답하기로 결정.
