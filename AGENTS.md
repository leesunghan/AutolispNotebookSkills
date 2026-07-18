# AGENTS.md

이 파일은 이 저장소(repository) 내에서 설명, AutoLISP, DCL 코드를 통합하여 포함하는 AutoLISP Notebook(`.lspnb`) 파일을 다룰 때 AI 코딩 에이전트(Claude Code, Cursor, Copilot, Antigravity 등)가 지켜야 할 가이드를 제공합니다.

## 저장소 개요 (Repository Overview)

Claude.ai, Claude Code 및 에이전트 워크플로우가 AutoLISP Notebook(`.lspnb`)을 빌드, 검토 및 유지 관리할 수 있도록 돕는 스킬(skills) 모음입니다. 이 노트북들은 마크다운(Markdown) 설명 셀과 실행 가능한 AutoLISP 코드 셀을 교대로 배치하여 학습과 자동화를 체계화합니다.

## OpenCode 및 AutoLISP Notebook 통합

이 환경은 `/skills` 디렉토리와 활성화된 에이전트 역할(personas)에 의해 작동하는 **스킬 기반 실행 모델(skill-driven execution model)**을 사용합니다.

### 핵심 규칙 (Core Rules)

1. **스킬 기반 실행**: 작업에 매핑되는 스킬(`skills/<스킬명>/SKILL.md`)을 먼저 열어 가이드를 엄격히 준수하십시오. 스킬을 거치지 않고 노트북 파일(`.lspnb`)을 직접 구현해서는 안 됩니다.
2. **프로젝트 격리**: 모든 신규 산출물(`SPEC.md`, `plan.md`, `todo.md`, `.lspnb` 등)은 반드시 `projects/<프로젝트명>/` 하위에 생성하고 격리 보존해야 합니다.
3. **노트북 셀 패턴 및 세분화**: 마크다운 설명 셀과 AutoLISP 코드 셀이 반드시 교대로 배치되어야 합니다. AutoLISP 노트북의 주목적은 목적 기능의 리습코드를 생성하고 코드를 학습하는 데 있으므로, 사용한 LISP 함수 및 로직을 설명하는 마크다운과 코드 셀을 더 미세하게 세분화합니다(예: 오류 처리 함수 `*error*` 선언부, 시스템 변수 백업부 등을 각각 개별 설명 셀과 코드 셀로 분리). 이때 코드 셀의 LISP 코드는 `(progn ...)`으로 묶지 않고 일반 LISP 코드 형태(예: `(setq a 1) (setq b 2)`)로 직접 작성합니다.
4. **노트북 포맷 및 이중 이스케이프**: `.lspnb` 파일은 루트 레벨 JSON 배열(`[...]`) 형식이며, `"source"` 필드는 단일 문자열이어야 합니다. 도구(API)를 통해 편집할 때 최종 JSON에 오류가 없도록 아래 문자 매핑 규칙을 철저히 준수하십시오.

   | 원래 소스 내용 (LISP/Markdown) | 최종 JSON 파일에 쓰여야 할 텍스트 | 도구 호출 인자(API Parameter)에 작성할 이중 이스케이프 |
   | :--- | :--- | :--- |
   | **LISP 코드 줄바꿈** | `\r\n` (CRLF 문자 표기) | `\\r\\n` (역슬래시 2개) |
   | **마크다운 줄바꿈** | `\n` (LF 문자 표기) | `\\n` (역슬래시 2개) |
   | **LISP 내부 문자열 따옴표 `"`** | `\"` (이스케이프된 따옴표) | `\\\"` (역슬래시 2개) |
   | **LISP 내부 백슬래시 `\`** | `\\` (두 개의 백슬래시) | `\\\\` (역슬래시 4개) |

   > [!IMPORTANT]
   > **절대적 금지 사항 (CRITICAL PROHIBITION)**: 도구 호출 인자 내부의 `"source"` 필드 값 안에 **실제 줄바꿈(Enter 입력으로 인한 physical newline)**을 절대 넣지 마십시오. 모든 개행은 문자열 내에서 반드시 `\\r\\n` 또는 `\\n` 문자 기호로만 표현되어 한 줄(Single Line)로 입력되어야 합니다. 실제 줄바꿈이 포함되면 파일에 제어 문자가 그대로 들어가 JSON 포맷 파싱 오류를 유발합니다.

5. **문서 한글화 표준**: 새로운 프로젝트나 세션을 시작할 때 생성되는 모든 사양서(`SPEC.md`), 계획서(`plan.md`/`implementation_plan.md`), 태스크 목록(`todo.md`/`task.md`)은 기본 템플릿의 형식을 따르되, 본문 내용은 반드시 한국어(Korean)로만 작성 및 유지되어야 합니다.
6. **마크다운 depth 구조화 표준**: 노트북 내의 마크다운 헤더는 구조적 가독성을 확보하기 위해 다음 3단계 depth 구분을 철저히 준수합니다.
   - `# 타이틀` (노트북 대제목)
   - `## [대분류]` 또는 `## [대분류] 소단계:` (예: `## [환경설정]`, `## [중심선 그리기] 1단계:`)
   - `### 소단계` (예: `### 1단계: 비상 ...`, `### 2단계: Undo ...` 등)

### 의도 → 스킬 매핑 (Intent → Skill Mapping)

에이전트는 사용자의 의도를 적절한 스킬에 자동으로 매핑해야 합니다:

- 노트북 기능 생성 → `spec-driven-development` 진행 후 `incremental-implementation` 및 `test-driven-development` 진행
- 계획 수립 / 작업 분할 → `planning-and-task-breakdown`
- AutoLISP 버그 / 실행 실패 / CAD 크래시 → `debugging-and-error-recovery`
- 노트북 코드 리뷰 → `code-review-and-quality`
- 리팩토링 / 단순화 → `code-simplification`
- CAD 자동화 설계 → `api-and-interface-design`

### 생명주기 매핑 (Lifecycle Mapping - 암묵적 명령어)

일반적인 웹 개발 주기 대신, 다음과 같은 AutoLISP Notebook 생명주기를 따르십시오:

- **DEFINE (정의)** → `spec-driven-development` (노트북 구조, 대상 CAD 플랫폼, 레이어, OSNAP 규칙 및 Undo 바운더리를 명시합니다).
- **PLAN (계획)** → `planning-and-task-breakdown` (셀 단위의 세로 슬라이스로 작업을 분할합니다).
- **BUILD (구현)** → `incremental-implementation` + `test-driven-development` (교차 셀을 세분화하여 작성하고, 개별 코드 셀의 `progn` 래핑을 배제한 채 변수 로컬화 및 OSNAP 제어가 포함되었는지 확인합니다).
- **VERIFY (검증)** → `debugging-and-error-recovery` (AutoLISP 실행 실패, 괄호 불일치, CAD 시스템 변수 상태 버그 등을 찾아내고 조치합니다).
- **REVIEW (검토)** → `code-review-and-quality` (변수 로컬화, ActiveX 객체 정리, CAD 환경 복원 상태를 점검합니다).
- **SHIP (배포)** → `shipping-and-launch` (JSON 구조를 검증하고, VS Code Notebook 확장 도구에서 오류 없이 로드되는지 확인한 뒤 테스트 실행을 마칩니다).

### 실행 모델 (Execution Model)

모든 요청에 대해 다음 단계를 수행하십시오:

1. 적용 가능한 스킬이 있는지 판단합니다.
2. 적절한 스킬을 호출합니다.
3. 워크플로우를 엄격히 따릅니다.
4. 필요한 사양 정의(spec) 및 계획(plan) 단계가 완전히 완료된 후에만 실제 구현으로 진행합니다.

### 자기합리화 금지 (Anti-Rationalization)

다음과 같은 생각은 잘못된 것이므로 철저히 무시해야 합니다:
- *"이 노트북 셀 코드는 너무 간단해서 스킬을 적용할 필요가 없다."*
- *"그냥 JSON 셀을 빠르게 작성하면 끝난다."*
- *"코드를 다 짠 다음에 CAD에서 한꺼번에 테스트하겠다."*

올바른 행동 방식:
- 항상 사양 정의, 계획, 검증을 단계별(incremental)로 수행하십시오. 각 셀은 실행된 후 반드시 CAD 환경을 안정적인 상태로 유지해야 합니다.

## 오케스트레이션: 페르소나, 스킬 및 커맨드

이 저장소는 다음과 같이 구성 가능한 3가지 계층으로 이루어져 있습니다:

- **스킬 (Skills)** (`skills/<name>/SKILL.md`) — 단계별 절차와 완료 조건을 포함한 워크플로우입니다. *방법(How)*에 해당합니다. 사용자의 의도가 일치할 때 반드시 거쳐야 하는 경로입니다.
- **페르소나 (Personas)** (`agents/<role>.md`) — 관점과 출력 포맷을 가진 에이전트 역할입니다. *담당자(Who)*에 해당합니다.
- **커맨드 (Commands)** — 에이전트 평가를 실행하는 조정기 역할을 합니다.

### 에이전트 페르소나 (Agent Personas)

- `code-reviewer.md` — 변수 로컬화, ActiveX 자원 정리, 노트북의 가독성에 중점을 두어 코드 리뷰를 수행합니다.
- `security-auditor.md` — CAD 안전 감사관(CAD Safety Auditor) 역할을 수행하여 CAD 프로세스 크래시, 무한 루프, 도면 데이터베이스 및 환경 변수 상태 오염을 방지합니다.
- `test-engineer.md` — 노트북 셀이 올바르게 실행되는지 테스트하고, 출력된 CAD 데이터베이스 객체를 검증합니다.
- `cad-performance-auditor.md` — 노트북 코드 실행 효율성(선택 세트 최적화, CMDECHO 억제, ActiveX 최적화)을 감사합니다.

## 새로운 스킬 만들기

스킬은 YAML 프론트매터(frontmatter)를 포함하여 `skills/<kebab-case-name>/SKILL.md` 경로에 작성합니다. 실행 가능한 CAD 자동화 도구 헬퍼를 함께 배포할 때만 `scripts/` 디렉토리를 추가하십시오.
