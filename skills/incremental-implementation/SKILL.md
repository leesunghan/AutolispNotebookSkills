---
name: incremental-implementation
description: AutoLISP Notebook (.lspnb) 파일의 변경 사양을 점진적으로 개발하고 배포합니다. 각 단계별로 셀 단위 개발을 할 때 활용하십시오.
---

# 점진적 기능 구현 (AutoLISP Notebook 에디션)

## 개요

AutoLISP Notebook을 짤 때는 얇고 명확한 세로 단위 슬라이스로 빌드하십시오. 하나의 마크다운 설명 셀과 AutoLISP 코드 셀 세트를 코드로 치고, 캐드에 올려 로드해 보고, 결과 도면의 무결성을 검증한 다음, 다음 단계 셀로 뻗어나가는 구조입니다. 매 개발 주기마다 노트북 파일의 JSON 구문 유효성을 유지하고 캐드 시스템의 원본 상태를 지켜야 합니다.

## 개발 루프 (The Increment Cycle)

```
┌──────────────────────────────────────────────┐
│                                              │
│   셀 세트 작성 ──→ CAD 로드/실행 ──→ 검증 ──┐ │
│        ▲                                    │ │
│        └──────────── 저장 및 백업 ◄─────────┘ │
│                      │                       │
│                      ▼                       │
│                  다음 셀 작성                │
│                                              │
└──────────────────────────────────────────────┘
```

매 단계마다 다음 4 단계를 순환하십시오:
1. **셀 세트 작성 (Write)**: 마크다운 가이드 설명 셀과 그에 해당하는 AutoLISP 코드 실행 셀을 개발합니다.
2. **CAD 로드/실행 (Load/Run)**: VS Code Notebook 확장을 통해 캐드와 통신하여 AutoLISP 블록을 실행시킵니다.
3. **검증 (Verify)**: 결과 객체들이 정상적으로 생성되었는지 관찰하고, 환경 변수가 온전히 돌아왔는지 점검하며, 에러 출력을 감시합니다.
4. **저장 및 백업 (Save/Commit)**: 유효한 작동 상태의 `.lspnb` 파일을 세이브하고 형상 제어 관리를 적용합니다.

---

## 노트북 코딩 및 코딩 스타일 규범 (Notebook Code & Style Rules)

노트북 내부의 AutoLISP 셀 코드를 작성할 때는 아래의 강제 규범을 무조건 지켜야 합니다:

### Rule 1: 언제나 코드 셀 내용을 `progn`으로 래핑할 것
노트북 환경은 하나의 셀 안에서 다중 실행 라인들을 처리하므로, 코드 전체가 원자성 있게 한 번에 컴파일되어 돌 수 있도록 전체 구문을 반드시 `(progn ...)` 블록으로 감싸야 합니다.
```json
{
  "id": "cell_02_layer_setup",
  "type": "code",
  "source": [
    "(progn\r\n",
    "  (vl-load-com)\r\n",
    "  (if (not (tblsearch \"LAYER\" \"S-WALL-CONC\"))\r\n",
    "      (command \"._layer\" \"_make\" \"S-WALL-CONC\" \"_color\" \"3\" \"\" \"\")\r\n",
    "  )\r\n",
    "  (princ \"\\n레이어 설정이 완료되었습니다.\")\r\n",
    "  (princ)\r\n",
    ")"
  ],
  "language": "autolisp"
}
```

### Rule 2: 전체 코드 셀 단위를 실행 취소(Undo) 마크로 묶을 것
사용자가 코드 실행 버튼을 누르고 난 후 언제라도 `Ctrl+Z`를 한 번 누르면 해당 코드 셀이 도면에 남긴 모든 변경 사항이 깔끔하게 한 번에 뒤로가기 취소되도록 제어해야 합니다.
```lisp
(progn
  ;; Undo 그룹 시작 마킹
  (command "._undo" "_group")
  
  ;; ... 셀 내부 드로잉 비즈니스 로직 작성 ...
  
  ;; Undo 그룹 종료 마킹
  (command "._undo" "_end")
  (princ)
)
```

### Rule 3: 에러 차단 및 실시간 환경 변수 복구 처리
작업 도중 에러가 나거나 사용자가 ESC를 마구 눌러 프로그램 작동을 중단시켰을 때 캐드 옵션을 망가진 채로 방치하면 안 됩니다. 기존의 시스템 변수를 무조건 먼저 안전하게 받아두고, 비상 탈출을 위한 전용 로컬 `*error*` 처리기를 세팅하십시오.
```lisp
(progn
  (command "._undo" "_group")
  ;; 1. 기존 시스템 변수 상태 임시 백업
  (setq old-osmode (getvar "OSMODE")
        old-cmdecho (getvar "CMDECHO")
        old-error *error*
  )
  
  ;; 2. 비상 예외 상황 대응용 로컬 에러 핸들러 구성
  (defun *error* (msg)
    (setvar "OSMODE" old-osmode)
    (setvar "CMDECHO" old-cmdecho)
    (command "._undo" "_end")
    (setq *error* old-error)
    (princ (strcat "\n비상 정지 처리 완료: " msg))
    (princ)
  )
  
  (setvar "CMDECHO" 0)
  
  ;; ... 드로잉 비즈니스 로직 실행 ...
  
  ;; 3. 정상 수행 성공 시의 복원 처리
  (setvar "OSMODE" old-osmode)
  (setvar "CMDECHO" old-cmdecho)
  (command "._undo" "_end")
  (setq *error* old-error)
  (princ "\n성공적으로 완료되었습니다!")
  (princ)
)
```

### Rule 4: OSNAP 제어 (선택 시점과 그리기 시점의 분리)
- **객체/지점 선택 시 (Select)**: 사용자가 기존 도면 요소를 마우스로 가리켜 정확한 지점을 스냅 잡을 수 있도록 원본 `OSMODE` 상태를 그대로 켜두어야 합니다.
- **프로그램 작도 시 (Draw)**: 계산된 정확한 수학적 기하 좌표대로 선이 그려지도록 `(setvar "OSMODE" 0)`을 적용하여 다른 끝점이나 근처점으로 강제 유도되는 오작동을 차단해야 합니다.

### Rule 5: 완벽한 JSON 문법 규격 준수
노트북 파일을 텍스트 형식으로 변경할 때 JSON 구조의 유효성이 절대로 깨지지 않도록 통제하십시오. AutoLISP 코드 내부의 문자열을 위해 사용한 큰따옴표는 소스 문자열 안에서 무조건 `\"`로 올바르게 이스케이프 처리되어야 하며, 줄바꿈은 `\r\n` 등으로 올바르게 치환되거나 문자열 배열의 별도 요소로 파싱되어야 합니다.

---

## 단계별 완료 체크리스트 (Increment Checklist)

하나의 셀 세트를 덧붙일 때마다 아래 사항을 셀프 체크하십시오:
- [ ] 생성하거나 수정한 `.lspnb` 파일이 `projects/<프로젝트명>/` 하위에 생성/수정되었는가?
- [ ] 생성하거나 수정한 `.lspnb` 파일이 문법적으로 유효한 JSON 형식인가?
- [ ] 작성한 AutoLISP 코드의 괄호 짝이 정확히 맞아 컴파일이 수행되는가?
- [ ] 실행 셀 내부가 단 하나의 `(progn ...)` 블록으로 깔끔하게 묶여 있는가?
- [ ] 셀 내부 실행을 전체적으로 묶어주는 undo 마킹 구문이 적용되어 있는가?
- [ ] 작동 에러나 중도 탈출(ESC) 시 시스템 변수가 원래대로 무조건 돌아가는가?
- [ ] 실제로 선을 그려넣는 드로잉 구문 실행 직전에만 OSNAP을 0으로 껐는가?

---

## 부록: AutoLISP Notebook (.lspnb) JSON 포맷 명세

에이전트는 노트북 파일을 작성하거나 편집할 때 아래의 JSON 구조와 규칙을 엄격하게 지켜야 합니다.

### 1. 기본 JSON 구조
`.lspnb` 파일은 하나의 루트 객체로 구성되며, 그 아래에 `cells`라는 이름의 배열을 포함합니다. 설명(Markdown), AutoLISP, DCL을 별도 파일로 쪼개지 않고, 이 단일 `.lspnb` 파일 내에 셀 단위로 통합하여 관리합니다.
```json
{
  "cells": [
    {
      "id": "unique_cell_id_1",
      "type": "markdown",
      "source": [
        "# 1. 벽체 중심선 선택 안내\n",
        "도면에서 벽체의 기준이 될 **중심선**을 마우스로 클릭하십시오."
      ]
    },
    {
      "id": "unique_cell_id_2",
      "type": "code",
      "language": "autolisp",
      "source": [
        "(progn\n",
        "  (vl-load-com)\n",
        "  (setq ent (entsel \"\\n중심선을 선택하십시오: \"))\n",
        "  (princ)\n",
        ")"
      ]
    }
  ]
}
```

### 2. 핵심 필드 규격
- **`cells`**: 셀 객체들의 배열.
- **`id`**: 개별 셀을 구분하는 고유 식별자 (소문자, 숫자, 언더바 권장).
- **`type`**: `"markdown"` 또는 `"code"` 중 하나여야 합니다.
- **`language`**: `type`이 `"code"`일 때 반드시 `"autolisp"`로 지정해야 합니다.
- **`source`**: 문자열의 배열. 각 배열 요소는 개행 문자(`\n` 또는 `\r\n`)를 포함하여 줄바꿈 단위로 쪼개어 기술됩니다.

### 3. 문자열 및 코드 이스케이프 주의사항
- AutoLISP 코드 내에서 프롬프트 메시지나 파일 경로에 큰따옴표를 사용할 때, JSON 문자열 구조가 깨지지 않도록 반드시 `\"` 형태로 더블 이스케이프 처리하십시오.
  - **잘못된 예**: `"source": ["(princ "\n완료")"]` (JSON 구문 에러 유발)
  - **올바른 예**: `"source": ["(princ \"\\n완료\")"]`
- 백슬래시(`\`) 역시 `\\` 형태로 바르게 처리되어야 캐드 환경에 로드되었을 때 `\n`(개행), `\t`(탭)로 올바르게 해석됩니다.

