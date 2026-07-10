# AGENTS.md

이 파일은 이 저장소(repository) 내에서 설명, AutoLISP, DCL 코드를 통합하여 포함하는 AutoLISP Notebook(`.lspnb`) 파일을 다룰 때 AI 코딩 에이전트(Claude Code, Cursor, Copilot, Antigravity 등)가 지켜야 할 가이드를 제공합니다.

## 저장소 개요 (Repository Overview)

Claude.ai, Claude Code 및 에이전트 워크플로우가 AutoLISP Notebook(`.lspnb`)을 빌드, 검토 및 유지 관리할 수 있도록 돕는 스킬(skills) 모음입니다. 이 노트북들은 마크다운(Markdown) 설명 셀과 실행 가능한 AutoLISP 코드 셀을 교대로 배치하여 학습과 자동화를 체계화합니다.

## OpenCode 및 AutoLISP Notebook 통합

이 환경은 `/skills` 디렉토리와 활성화된 에이전트 역할(personas)에 의해 작동하는 **스킬 기반 실행 모델(skill-driven execution model)**을 사용합니다.

### 핵심 규칙 (Core Rules)

- 작업(task)이 특정 스킬과 일치하면, 반드시(MUST) 해당 스킬을 호출해야 합니다.
- 스킬들은 `skills/<skill-name>/SKILL.md`에 위치합니다.
- 상응하는 스킬을 따르지 않고 노트북 파일(`.lspnb`)을 직접 구현해서는 안 됩니다.
- 모든 프로젝트 관련 신규 산출물(`SPEC.md`, `plan.md`, `todo.md`, `.lspnb` 노트북 파일 등)은 루트 디렉터리를 어지럽히지 않도록 반드시 `projects/<프로젝트명>/` 하위 경로에 생성되고 격리 보존되어야 합니다.
- 다음과 같이 셀이 교대로 나타나는 패턴을 강제해야 합니다: **마크다운 셀(설명) ──→ AutoLISP 코드 셀(progn으로 래핑) ──→ 마크다운 셀 ──→ AutoLISP**.
- **개행 문자 및 노트북 포맷 규정**: 모든 `.lspnb` 노트북 파일은 루트 레벨의 JSON 배열(`[...]`) 형식이어야 하며, 각 셀 내의 `"source"` 속성은 문자열 배열이 아닌 **단일 문자열(String)** 형식이어야 합니다. 코드 셀 내의 줄바꿈 개행 기호는 반드시 Windows 방식인 `\r\n` (CRLF)이 포함된 단일 문자열로 작성되어야 하며, 마크다운 셀 내의 줄바꿈은 `\n`이 포함된 단일 문자열로 작성되어야 합니다.
  
  **올바른 포맷 예시:**
  ```json
  [
    {
      "id": "xrikqmefw_mrfbt2ve",
      "type": "markdown",
      "source": "# New AutoLISP Notebook\nDouble-click to edit.",
      "language": "markdown"
    },
    {
      "id": "vs9w062jt_mrfbt2ve",
      "type": "code",
      "source": "(princ \"Hello World\") \r\n;;;\r\n;;;",
      "language": "autolisp"
    }
  ]
  ```


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
- **BUILD (구현)** → `incremental-implementation` + `test-driven-development` (교차 셀을 작성하고, `progn` 래핑, 변수 로컬화, OSNAP 제어가 포함되었는지 확인합니다).
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
