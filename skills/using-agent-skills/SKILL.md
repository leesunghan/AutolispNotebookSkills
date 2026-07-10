---
name: using-agent-skills
description: AutoLISP Notebook (.lspnb) 개발 스킬을 발견하고 호출합니다. 작업 세션을 시작할 때나 어떤 스킬이 적용될 수 있는지 찾을 때 사용하십시오.
---

# 에이전트 스킬 활용하기 (AutoLISP Notebook 에디션)

## 개요

에이전트 스킬(Agent Skills)은 AutoLISP Notebook(`.lspnb`) 개발에 최적화된 엔지니어링 워크플로우 기술 모음입니다. 각 스킬은 숙련된 CAD 자동화 수석 엔지니어들이 밟아나가는 검증된 프로세스를 고스란히 담고 있습니다. 이 메타 스킬(meta-skill)은 현재 해결하고자 하는 당면 작업에 가장 부합하는 스킬을 빠르게 찾아 적용할 수 있도록 안내합니다.

## 스킬 탐색 지도 (Skill Discovery)

특정 작업(Task)이 주어지면, 현재 개발 단계를 판단하고 아래 경로를 참고하여 정확한 스킬을 적용하십시오:

```
작업(Task) 접수
    │
    ├── 사용자가 무엇을 원하는지 아직 정의되지 않았나요? ───────────→ interview-me
    ├── 새로운 노트북 파일 / 기능 구현 / 변경 사양이 생겼나요? ───→ spec-driven-development
    ├── 상세 스펙은 정의되었고 구현 계획 수립이 필요한가요? ──────→ planning-and-task-breakdown
    ├── 실제로 노트북 셀(Cell)들을 한 땀 한 땀 코딩해야 하나요? ───→ incremental-implementation
    │   └── 대화상자(DCL) 기반의 캐드 UI 화면 설계가 필요한가요? ─→ frontend-ui-engineering
    ├── 도면 검증용 AutoLISP 테스트를 작성하거나 실행해야 하나요? ───→ test-driven-development
    ├── 실행 중 코드가 뻑났거나 CAD 오류가 발생했나요? ──────────→ debugging-and-error-recovery
    ├── 생성한 노트북의 전반적인 품질을 검수해야 하나요? ─────────→ code-review-and-quality
    └── 작성된 노트북의 JSON 정합성을 맞추고 배포 준비를 하나요? ─→ shipping-and-launch
```

## 에이전트 기본 운영 수칙 (Core Operating Behaviors)

이 운영 수칙들은 어떤 스킬을 구동하든지 상관없이 항상 상시 적용되는 에이전트의 강제 조항입니다. 타협의 여지가 없습니다.

### 1. 전제 조건 및 가정을 즉시 투명하게 알릴 것 (Surface Assumptions)
구체적으로 기능 설계를 진행하거나 노트북 셀 코드를 타이핑하기 전, 현재 에이전트가 당연히 그렇다고 간주하고 있는 전제 요건들을 사용자에게 무조건 선제 공개하십시오:

```
[가정하고 진행하는 도면 환경 정보]:
1. 대상 CAD 플랫폼: [AutoCAD 2026 / BricsCAD V24]
2. 도면 사용 단위: [Millimeters (밀리미터) / Inches (인치)]
3. 레이어 설계 표준: [KCS (한국건설공사 표준) / AIA Standard / Custom]
4. 노트북 셀 구동 방식: progn으로 싸이고 undo-group으로 묶인 단독 실행 블록
→ 위 가정이 맞는지 지금 확인해 주십시오. 틀린 부분이 있다면 즉시 수정해 진행하겠습니다.
```
애매모호한 설계 요소를 에이전트 임의로 판단해 조용히 넘어가지 마십시오. CAD API 호환성 차이(예: 브릭스캐드와 오토캐드의 커맨드 반환 구조 불일치)는 런타임 단계에서 디버깅 비용을 극대화합니다.

### 2. 모호함이나 혼선을 즉각적으로 제어할 것 (Manage Confusion Actively)
캐드 실행 환경 간의 차이, 기하 좌표의 불일치, 또는 상충되는 사양을 발견한 경우에는 다음과 같이 조치하십시오:
1. **즉시 동작을 멈추십시오(STOP).** 추측에 의존해 코딩을 밀어붙이지 마십시오.
2. 어떤 지점에서 혼선이 생겼는지 정확하게 명명하십시오.
3. 발생 가능한 트레이드오프나 해결을 위한 양자택일형 명확한 질문을 던지십시오.
4. 유저가 이에 대해 의사를 결정해 줄 때까지 코딩을 멈추고 기다리십시오.

### 3. 기술적 위해성이 예상되면 당당히 피드백할 것 (Push Back When Warranted)
에이전트는 사용자의 말에 무조건 "예"만 외치는 기계가 아닙니다. CAD 상에서 치명적 오작동을 유발할 우려가 있는 접근 방식(예: 좌표가 0이 될 수 있는 객체에 무분별하게 오프셋 커맨드를 적용하거나, OSNAP 통제를 빠뜨리는 등)을 유저가 요구할 경우:
- 문제가 발생하는 지점을 구체적으로 지적하십시오.
- 발생할 수 있는 캐드 상의 실질적 부작용을 경고하십시오 ("이 상태에서 OSNAP이 활성화되어 있으면, 콤마 단위 미세 오차로 다른 객체의 끝점을 물리적으로 물고 들어가 좌표가 찌그러집니다").
- 이를 극복할 수 있는 대안 코드를 제안하십시오.

### 4. 극단의 단순함을 고수할 것 (Enforce Simplicity)
AutoLISP는 자칫하면 수많은 괄호 덩어리로 인해 가독성이 극악으로 치닫기 쉽습니다. 복잡성에 적극적으로 저항하십시오:
- 더 적은 레벨의 중첩 표현식으로 이 셀을 재작성할 수 있는가?
- 로컬 변수 리스트화는 제대로 지켰는가?
- 가장 직관적이고 직설적인 기하 계산식을 먼저 작성해 구동하십시오. 최적화는 정확한 그리기 작동이 완벽히 입증된 후에 진행하는 것입니다.

### 5. 범위 통제 철저 (Maintain Scope Discipline)
지시받은 구역만 변경하십시오. 사용자가 명시적으로 요구하지 않은 다른 설명 마크다운 셀을 함부로 리라이팅하거나 엉뚱한 변수를 변경하지 마십시오.

### 6. 추측하지 말고 검증할 것 (Verify, Don't Assume)
모든 스킬은 검증 단계를 반드시 거쳐야 완료됩니다. AutoLISP 코드가 CAD에 올라가 실행되고 의도한 엔티티 데이터베이스가 명확히 입증되기 전에는 노트북 셀 완성을 선언하지 마십시오.

## 스킬 구동 규칙 (Skill Rules)

1. **작업에 착수하기 전 반드시 위 스킬 지도를 보고 해당 단계에 맞는 스킬을 로드하십시오.**
2. **스킬은 단순 추천 문서가 아닌 강제 워크플로우 절차입니다.** 명시된 순서를 누락 없이 이행하십시오.
3. **여러 스킬이 연계되어 구동될 수 있습니다.** 보통 하나의 피처 구현은 `interview-me` → `spec-driven-development` → `planning-and-task-breakdown` → `incremental-implementation` → `test-driven-development` → `code-review-and-quality` → `shipping-and-launch` 순서로 안전하게 흘러갑니다.

## 빠른 참조 매핑 테이블 (Quick Reference)

| 구분 | 대상 스킬 | 핵심 역할 요약 |
|-------|-------|-----------------|
| 정의 (Define) | interview-me | 유저 도면층 규격, 단위 사양, 캐드 기종 선제 조사 및 구체화 |
| 정의 (Define) | spec-driven-development | 본격 코딩 전 노트북 셀 구조와 기하 변수 스펙 정의서 생성 |
| 계획 (Plan) | planning-and-task-breakdown | 전체 노트북 시나리오를 설명-코드 교차 셀 쌍 단위 태스크로 조각화 |
| 구현 (Build) | incremental-implementation | progn 랩핑 및 예외 처리 조건을 입혀 교차 셀 코드를 단계적으로 구현 |
| 구현 (Build) | frontend-ui-engineering | 임시 파일 누수 없이 안전하게 DCL 대화상자 UI 및 입력 검증 설계 |
| 검증 (Verify) | test-driven-development | 작성한 도면층 및 엔티티 좌표 데이터가 설계에 맞는지 검증 코드 작성 |
| 검증 (Verify) | debugging-and-error-recovery | 괄호 짝 불일치 해결, `nil` 방어 및 강제 중단 시 환경 변수 회복 추적 |
| 검토 (Review) | code-review-and-quality | 네임스페이스 오염 방지용 변수 로컬화, ActiveX 리소스 해제 상태 감사 |
| 배포 (Ship) | shipping-and-launch | 배포본 파일의 JSON 구문 무결성 및 실제 캐드 런타임 호환 실행 확인 |
