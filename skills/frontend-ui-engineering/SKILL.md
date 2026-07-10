---
name: frontend-ui-engineering
description: AutoLISP Notebook (.lspnb) 환경에서 DCL (Dialog Control Language)을 활용하여 표준화되고 안전한 캐드 대화상자(UI)를 구성합니다.
---

# 프론트엔드 UI 엔지니어링 (DCL & AutoLISP Notebook 에디션)

## 개요

AutoCAD 및 BricsCAD 환경에서 작동하는 사용자 인터페이스(UI)를 **DCL (Dialog Control Language)**을 사용하여 고품질로 구축합니다. 본 스킬 가이드의 목표는 AI가 무작위로 생성해 정렬이 어긋나거나, 사용자 취소 시 캐드가 일시적으로 멈추거나(Freeze), 파일 시스템에 임시 DCL 파일이 지워지지 않고 남는 등의 결함을 방지하는 것입니다. 

사용자가 조작을 직관적으로 이해할 수 있는 세련된 배치와 정렬, 견고한 상태 관리, 그리고 완벽한 예외/자원 정리 로직을 갖춘 프로덕션 등급의 CAD UI 설계를 지향합니다.

## 사용 시점

- 대화상자(Dialog Box)를 통해 사용자로부터 여러 수치나 레이아웃 매개변수를 한 번에 입력받아야 할 때.
- 그리기 작업 전, 체크박스(Toggles)나 라디오 버튼(Radio Buttons), 드롭다운(Popup Lists) 등으로 옵션을 선택받고자 할 때.
- 기존 DCL 인터페이스의 레이아웃 어긋남, 데이터 유효성 검사 누락, 또는 다이얼로그 강제 종료 버그를 수정할 때.

---

## 1. DCL & Notebook 통합 아키텍처 (DCL & Notebook Integration)

AutoLISP Notebook(`.lspnb`) 환경은 단일 파일 통합 지향성을 가집니다. 따라서 DCL 파일을 외부에 별도로 두는 대신, **LISP 코드 셀 내에서 임시 DCL 파일을 동적으로 생성하고, 실행 후 즉시 제거하는 인라인 템프 DCL 패턴(Inline Temp DCL Pattern)**을 무조건 사용해야 합니다.

### 인라인 DCL 수명주기 템플릿 (DCL Lifecycle Template)

모든 DCL UI 실행 코드 셀은 자원 누수를 차단하기 위해 반드시 아래 구조를 기본 뼈대로 사용해야 합니다.

```lisp
(progn
  ;; 1. 환경 변수 백업 및 임시 DCL 경로 할당
  (command "._undo" "_group")
  (setq old-osmode (getvar "OSMODE")
        old-cmdecho (getvar "CMDECHO")
        old-error *error*
        dcl-file (vl-filename-mktemp "ui-temp" "" ".dcl")
        dcl-id nil
  )

  ;; 2. 비상 에러 핸들러 구성 (DCL 해제 및 임시 파일 삭제 보장)
  (defun *error* (msg)
    (if (and dcl-id (> dcl-id 0))
      (unload_dialog dcl-id)
    )
    (if (and dcl-file (findfile dcl-file))
      (vl-file-delete dcl-file)
    )
    (setvar "OSMODE" old-osmode)
    (setvar "CMDECHO" old-cmdecho)
    (command "._undo" "_end")
    (setq *error* old-error)
    (princ (strcat "\n[에러] UI 실행 실패: " msg))
    (princ)
  )

  ;; 3. DCL 코드 내용 스트림 쓰기
  (setq fp (open dcl-file "w"))
  (write-line "grid_setup_dialog : dialog {" fp)
  (write-line "  label = \"그리드 작도 설정\";" fp)
  (write-line "  : column {" fp)
  (write-line "    : edit_box {" fp)
  (write-line "      key = \"grid_spacing\";" fp)
  (write-line "      label = \"그리드 간격 (mm): \";" fp)
  (write-line "      edit_width = 10;" fp)
  (write-line "    }" fp)
  (write-line "    ok_cancel;" fp)
  (write-line "  }" fp)
  (write-line "}" fp)
  (close fp)

  ;; 4. 다이얼로그 로드 및 초기화
  (setq dcl-id (load_dialog dcl-file))
  (if (not (new_dialog "grid_setup_dialog" dcl-id))
    (exit) ; 에러 발생 시 *error* 핸들러로 유도
  )

  ;; 5. 초기 입력 기본값 설정
  (set_tile "grid_spacing" "1000")

  ;; 6. 이벤트 콜백 바인딩
  (action_tile "accept" "(done_dialog 1)")
  (action_tile "cancel" "(done_dialog 0)")

  ;; 7. 다이얼로그 시작 및 리턴값 수집
  (setq dialog-status (start_dialog))

  ;; 8. 정상 수행 이후 자원 해제 및 임시 파일 청소
  (unload_dialog dcl-id)
  (setq dcl-id nil)
  (vl-file-delete dcl-file)

  ;; 9. 다이얼로그 결과 확인 후 드로잉 로직 실행
  (if (= dialog-status 1)
    (progn
      (setq final-spacing (atof (get_tile "grid_spacing")))
      (princ (strcat "\n선택된 간격: " (rtos final-spacing 2 0) "mm"))
      ;; ... 실물 드로잉 로직 연계 실행 ...
    )
    (princ "\n사용자가 작도를 취소했습니다.")
  )

  ;; 10. 원래 환경 복원
  (setvar "OSMODE" old-osmode)
  (setvar "CMDECHO" old-cmdecho)
  (command "._undo" "_end")
  (setq *error* old-error)
  (princ)
)
```

---

## 2. DCL 레이아웃 설계 표준 (DCL Layout Standards)

정돈되지 않은 UI는 가독성을 떨어뜨리고 사용자에게 혼란을 줍니다. 다음 규칙들을 지켜 세련된 UI를 설계하십시오.

### 2.1 행(Row)과 열(Column)의 명확한 구조화
- `row`와 `column` 타일을 사용해 요소들을 논리적 그룹으로 묶으십시오.
- 정보에 명확한 테두리가 필요할 때는 `boxed_row` 및 `boxed_column`을 적용하고 간결한 `label`을 지정하십시오.

### 2.2 고정 폭(fixed_width)과 라벨 정렬
- 컨트롤(입력창, 드롭다운 등)의 크기가 우스꽝스럽게 좌우로 길게 늘어지지 않도록 `edit_width`나 `width` 속성을 명확히 선언하십시오.
- 여러 입력창의 라벨을 세로로 나란히 정렬하기 위해, `column` 내의 각 `edit_box` 라벨 길이를 공백을 통해 유사하게 맞추거나 `fixed_width = true;`를 적용하십시오.

### DCL 속성 조합 가이드라인

| 타일 유형 (Tile Type) | 핵심 사용 속성 | 설계 유의사항 |
| :--- | :--- | :--- |
| **`edit_box`** | `key`, `label`, `edit_width`, `allow_accept = true;` | 사용자가 값 입력 후 엔터를 쳤을 때 다이얼로그가 곧바로 닫히도록 하려면 `allow_accept = true;`를 선언하십시오. |
| **`popup_list`** | `key`, `label`, `width`, `edit_width` | 드롭다운 목록이며, 로드 시 `start_list` 및 `add_list` API로 AutoLISP 단에서 아이템 리스트를 파퓰레이트해야 합니다. |
| **`toggle`** | `key`, `label` | 체크박스 형태. 상태값은 `"0"`(선택 해제) 또는 `"1"`(선택됨) 스트링 타입으로 리턴됩니다. |
| **`radio_button`** | `key`, `label` | 반드시 `radio_column` 또는 `radio_row` 내에 자식 타일들로 배치하여 독립 선택 관계를 구성하십시오. |

---

## 3. UI 상태 관리 및 데이터 유효성 검사 (State & Validation)

사용자 입력 데이터는 언제든 잘못된 값이 인입될 수 있습니다. (예: 숫자가 들어와야 할 곳에 문자가 적히거나, 음수 벽 두께가 들어오는 등). 작도 연산 시작 전에 완벽한 데이터 검증 필터를 구성해야 캐드 다운이나 오작동을 원천 봉쇄할 수 있습니다.

### 3.1 DCL 콜백 설계 원칙
- `action_tile` 내부의 AutoLISP 구문은 되도록 짧게 유지하십시오.
- 복잡한 검증 로직이 수반될 경우, 해당 동작을 처리할 **이벤트 핸들러 함수**를 AutoLISP 단에 정의하고 해당 함수명만 콜백 문자열로 지정하십시오.

### 3.2 실시간 입력 검증 예제 코드
아래 예제는 사용자가 입력창에 숫자가 아닌 문자를 입력하거나 범위 밖의 값을 입력했을 때, 다이얼로그 종료를 보류하고 에러 메시지를 노출하는 견고한 유효성 검증 기법입니다.

```lisp
;; 실시간 데이터 유효성 검증 핸들러
(defun validate-spacing-input ()
  (setq raw-val (get_tile "grid_spacing")
        num-val (atof raw-val)
  )
  (cond
    ;; 1. 공백 입력 검사
    ((= raw-val "")
     (set_tile "error" "에러: 그리드 간격을 입력하십시오.")
     (mode_tile "grid_spacing" 2) ; 해당 입력 필드로 포커스 복원
    )
    ;; 2. 문자 포함 검사 (atof 반환값이 0.0이고 원래 입력이 "0"이 아닌 경우)
    ((and (= num-val 0.0) (/= raw-val "0") (/= raw-val "0.0"))
     (set_tile "error" "에러: 유효한 숫자를 입력하십시오.")
     (mode_tile "grid_spacing" 2)
    )
    ;; 3. 하한값 제한 검사 (간격은 최소 100 이상이어야 함)
    ((< num-val 100.0)
     (set_tile "error" "에러: 간격은 최소 100mm 이상이어야 합니다.")
     (mode_tile "grid_spacing" 2)
    )
    ;; 4. 검증 통과 시 다이얼로그 정상 종료
    (T
     (set_tile "error" "") ; 에러창 클리어
     (done_dialog 1)       ; 1을 반환하며 다이얼로그 정상 종료
    )
  )
)

;; 다이얼로그 호출 세션 내부 적용 방식
(action_tile "accept" "(validate-spacing-input)")
```

DCL 하단에 `: text { key = "error"; value = ""; foreground = "red"; }`와 같이 에러 출력용 전용 문자 필드를 설계해 두면, 시스템 창을 띄우는 `(alert)` 호출보다 훨씬 자연스럽고 고급스러운 오류 알림 기능을 구현할 수 있습니다.

---

## 4. UI 안티패턴 방지 (Anti-Patterns to Avoid)

| 결함 유형 | 문제점 | 해결 방안 |
| :--- | :--- | :--- |
| **임시 `.dcl` 파일 방치** | `(vl-filename-mktemp)`로 생성된 파일들이 사용자 임시 디렉토리에 계속 쌓여 용량을 차지함. | 코드 블록의 성공 탈출 구문 및 `*error*` 핸들러 양측에 무조건 `(vl-file-delete dcl-file)`을 선언하십시오. |
| **다이얼로그 갇힘 현상 (Freeze)** | AutoLISP 에러로 동작이 끊기면 화면에 대화상자가 닫히지 않고 그대로 굳어 강제 종료해야 함. | `dcl-id` 검출 시 `(unload_dialog dcl-id)`가 `*error*` 핸들러에 필수 등록되어 있는지 재차 감수하십시오. |
| **정렬되지 않은 레이아웃** | 너비 제어가 되지 않은 입력 필드들이 양옆으로 과도하게 찢어지거나 오밀조밀하게 겹침. | `edit_width`와 `width`를 명시하여 컴포넌트의 가시적 배율을 정밀 조율하십시오. |
| **라디오 버튼 중복 선택** | 동일 성격의 옵션들이 중복 체크되거나 해제되지 않음. | 라디오 버튼 그룹은 반드시 `radio_row`나 `radio_column`으로 감싸 분리하십시오. |

---

## 5. DCL UI 엔지니어링 체크리스트

노트북 내에 DCL UI 셀을 통합 개발할 때 아래 조건의 충족 여부를 확인하십시오.

- [ ] 노트북 파일(`.lspnb`)의 JSON 구조가 이중 이스케이프 문자(`\"`, `\\`) 등으로 인해 훼손되지 않는가?
- [ ] DCL 파일 생성을 위한 스트림 작성 코드(`(open ... "w")` 및 `(close ...)`)가 짝을 이루고 잘 닫히는가?
- [ ] 다이얼로그 로드 실패 시 캐드 런타임 크래시가 나지 않도록 `(if (not (new_dialog ...)))` 방어 조치가 되어 있는가?
- [ ] 임의의 LISP 예외나 사용자의 강제 ESC 취소 발생 시에도 `*error*` 핸들러가 DCL 대화상자를 안전하게 종료(`unload_dialog`)하고 임시 DCL 파일을 즉시 삭제하는가?
- [ ] 입력값 검증용 이벤트 핸들러가 구성되어 부적절한 사용자 데이터 유입을 사전에 차단하는가?
- [ ] 모든 컨트롤 요소에 직관적인 `key`와 눈이 편안한 레이아웃 폭(`edit_width`)이 정의되어 있는가?
- [ ] 실행 완료 시 도면에 남은 UI 찌꺼기 파일이 깨끗이 삭제되었는가?
