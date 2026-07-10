---
name: spec-driven-development
description: 코딩을 시작하기 전에 AutoLISP Notebook (.lspnb) 사양 명세서(Spec)를 작성합니다. 신규 노트북 작성, AutoLISP 자동화 구현, 혹은 규모가 큰 캐드 기능 추가 시 활용하십시오.
---

# 스펙 기반 개발 (AutoLISP Notebook 에디션)

## 개요

AutoLISP 코드를 단 한 줄이라도 치기 전에 반드시 구조화된 스펙 문서(사양 명세서)를 기술하십시오. 이 스펙 문서는 사용자와 에이전트 간의 가장 확실한 합의점이자 단일 진실 공급원(Source of Truth)이 됩니다. 어떤 캐드 자동화를 달성할 것인지, 어떤 제도 표준을 지킬 것인지, 노트북의 셀이 어떤 순서로 배치될 것인지를 사전에 조율합니다.

## 사용 시점

- 설명, AutoLISP, DCL을 통합적으로 포함하는 새로운 AutoLISP Notebook(`.lspnb`) 파일 작성을 시작할 때.
- 도면 유닛(Unit), 형상 치수, 레이어 이름 표준 및 캐드 플랫폼 호환성 범위가 불확실할 때.
- 목표 작업이 다단계 기하 연산(예: 선 그리기 -> 평행 오프셋 -> 교차 부위 정리 -> 해치 영역 채우기)을 필요로 할 때.
- 도면 데이터베이스 내부 상태를 실제로 수정하는 드로잉 동작을 수행할 때.

**사용 불가능한 시점:** AutoLISP의 단순 한 줄 오타 수정, 변수 이름 변경 또는 단순한 텍스트 가독성 개선 작업 등.

## 게이트 통제형 워크플로우 (The Gated Workflow)

스펙 기반 개발은 4가지 고유 단계를 따릅니다. 이전 단계에 대해 사용자의 명시적인 승인(동의)을 득하기 전에는 결코 다음 단계로 넘어가지 마십시오.

```
스펙 작성 (SPECIFY) ──→ 구현 계획 (PLAN) ──→ 세부 태스크 (TASKS) ──→ 실제 코딩 (IMPLEMENT)
       │                       │                      │                     │
       ▼                       ▼                      ▼                     ▼
  사용자 검토             사용자 검토            사용자 검토           사용자 검토
   및 동의                 및 동의                및 동의               및 동의
```

### Phase 1: Specify (스펙 작성)

대략적인 큰 방향성부터 구체화하십시오. 사용자와 질답을 거치며 노트북 내에 탑재할 AutoLISP 그리기 규칙과 셀별 설계 구조를 확실히 매듭짓습니다.

**전제 요건(Assumptions) 선제 공개:** 스펙 문서를 적기 전, 본인이 임의로 전제하고 있는 캐드 세팅 정보를 일목요연하게 보고하십시오:

```
[가정하고 진행하는 도면 환경 정보]:
1. 대상 CAD 플랫폼: BricsCAD V24 (표준 접두사 사용)
2. 도면 사용 단위: Millimeters (1 unit = 1mm)
3. 레이어 설계 표준: 구조용 콘크리트 벽체는 S-WALL-CONC, 내부 마감선은 A-WALL-FINS
4. OSNAP 제어 규칙: 오작동을 예방하기 위해 좌표를 계산하여 그리는 매 순간 OSMODE를 0으로 제어함
→ 위 가정이 맞는지 지금 확인해 주십시오. 틀린 부분이 있다면 즉시 수정해 진행하겠습니다.
```

**그 다음, 아래 6가지 핵심 항목을 충족하는 스펙 문서를 작성하십시오:**

1. **Objective (목적)** — 어떤 작업을 자동화하려 하는가? 기존의 수동 제도 방식 중 어떤 수고를 덜어내는가? 타겟 CAD 플랫폼은 무엇인가?
2. **CAD Commands & APIs (명령어 및 API)** — 호출할 기본 커맨드(예: `._line`, `._offset`, `._fillet`) 목록과 활용할 ActiveX 인터페이스(예: `vla-addline`, `vlax-ename->vla-object`)를 기술합니다.
3. **Notebook Cell Structure (노트북 셀 배치 시나리오)** — 마크다운 설명 셀과 AutoLISP 코드 실행 셀이 교대로 배치될 전체 시나리오 구조를 사전에 기획합니다.
4. **Code Style & Safety (코드 안전 규칙)** — 변수 로컬화 범위, Undo 그룹화 범위, 드로잉 동작 도중의 OSNAP 제어 방식을 못박아둡니다.
5. **Testing Strategy (테스트 및 검증 전략)** — 작성된 도면 데이터베이스 상태가 온전한지 테스트할 AutoLISP 질의 방식(예: `tblsearch`를 통한 레이어 확인, `ssget`을 통한 그린 개체 체크)을 규정합니다.
6. **Boundaries (설계 경계선)**:
   - **Always do (반드시 지킬 조항):** 실패/중도 취소 시 무조건 `OSMODE` 및 `*error*`를 복원할 것, 모든 코드 셀은 `progn`으로 감쌀 것.
   - **Ask first (선제 조율 조항):** 요구 범위 밖의 사용자 정의 시스템 변수를 임의로 조정할 때, 도면에 이미 그려진 외부 엔티티를 삭제할 때.
   - **Never do (절대 금지 조항):** 셀 구동 후 임시 해제한 OSNAP을 원래대로 돌려놓지 않고 끝내기, 개체 선택 리턴 값에 대해 `nil` 방어 없이 바로 데이터 추출하기.

**스펙 작성 템플릿:**

```markdown
# Spec: [노트북 기능 명칭]

## Objective (개발 목적)
[이 노트북이 자동화하고자 하는 작업의 사양을 기술합니다. 타겟 사용자군 및 CAD 구동 플랫폼 명시.]

## Tech Stack & CAD Environment (캐드 구동 환경 및 기술 사양)
- 대상 파일: [projects/<프로젝트명>/notebook_name.lspnb]
- 대상 CAD 엔진: [AutoCAD / BricsCAD / 호환 CAD]
- 언어: AutoLISP (ActiveX 기능 활성화)
- 제도 단위: [Millimeters / Meters / Inches 등]

## CAD Commands & APIs Used (활용 명령어 및 API 세트)
- 내장 커맨드: [예: ._line, ._offset]
- ActiveX 인터페이스: [예: vla-AddLine, vla-Offset]

## Notebook Cell Structure (노트북 셀 시나리오 기획)
- Cell 1 (Markdown): [1단계 작업 안내 설명 작성]
- Cell 2 (AutoLISP): [1단계 동작을 수행할 progn 랩핑 코드, undo group 범위 내포]
- Cell 3 (Markdown): [2단계 작업 안내 설명 작성]
- Cell 4 (AutoLISP): [2단계 동작을 수행할 progn 랩핑 코드]

## Code Style & Safety Guidelines (안정성 확보 규범)
- 변수 로컬화: `(defun ... ( / var1 var2 ))` 형식의 함수 내부 로컬 처리 보장
- Undo 관리: 각 실행 코드 셀 단위의 완전한 Undo 묶음 적용
- OSNAP 통제: 점 지정/입력 시에는 켜두고, 내부 계산 좌표로 그릴 때는 임시 비활성화 처리

## Testing & Verification Strategy (동작 검증 방안)
- [작성 완료 후 레이어 구성 및 엔티티 생성 정합성을 체크할 검증 쿼리 구문 정의]

## Boundaries (설계 경계 조항)
- Always: [...]
- Ask first: [...]
- Never: [...]

## Success Criteria (최종 성공 기준)
- [예: "설계한 레이어가 정확히 생성되었고, 벽의 교차 구간 모따기선이 교차 오차 없이 안전하게 드로잉됨"]
```

---

### Phase 2: Plan (구현 계획 수립)
노트북 셀 배치 시나리오, 작성할 공통 AutoLISP 유틸리티 함수, 그리고 처리해야 할 복잡한 형상 벡터 기하 수학 공식의 해결 절차를 체계적으로 나열합니다.

### Phase 3: Tasks (세부 태스크 분할)

수립한 계획을 바탕으로 세부적인 셀 단위 구현 조각들로 태스크를 생성합니다:
```markdown
- [ ] Task: Cell 1 & 2 구현 (레이어 자동 생성 및 준비 단계)
  - 인수 조건: 구동 시 S-WALL-CONC 레이어가 초록색으로 자동 로드되어 있어야 함
  - 검증: `(tblsearch "LAYER" "S-WALL-CONC")` 실행 결과 레이어 정보 리스트가 출력됨
  - 대상 파일: [projects/<프로젝트명>/notebook.lspnb]
```

### Phase 4: Implement (단계별 코딩)
태스크 순서에 맞춰 `incremental-implementation` 및 `test-driven-development` 스킬에 근거해 차례대로 코드를 조립해 나갑니다.
