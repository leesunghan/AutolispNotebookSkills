# 테스트 패턴 참조 가이드 (AutoLISP 노트북 에디션)

AutoCAD/BricsCAD AutoLISP 노트북(`.lspnb`) 환경에서 테스트를 작성, 실행 및 검증하기 위한 퀵 참조 가이드입니다. 이 문서는 `test-driven-development` 스킬과 함께 참고하십시오.

---

## 목차

- [CAD TDD 사이클 (RED -> GREEN -> REFACTOR)](#cad-tdd-사이클)
- [CAD 데이터베이스 어설션](#cad-데이터베이스-어설션)
- [시스템 변수 검증](#시스템-변수-검증)
- [CAD Prove-It 패턴 (버그 수정)](#cad-prove-it-패턴)
- [테스트 격리 및 정리](#테스트-격리-및-정리)

---

## CAD TDD 사이클

CAD 자동화에서 테스트는 도면 데이터베이스(DWG database)의 상태를 직접 검사하는 실행 가능한 AutoLISP 셀 형태로 구현됩니다.

```lisp
;; 1. RED (실패하는 어설션): 기하학적 요소가 아직 도면에 존재하지 않으므로 검증이 실패하는지 확인
(progn
  (setq passed T)
  (if (not (tblsearch "LAYER" "A-WALL"))
      (progn (princ "\n[FAIL] A-WALL 레이어가 없습니다.") (setq passed nil))
  )
  (if (not (ssget "_X" '((0 . "LINE") (8 . "A-WALL"))))
      (progn (princ "\n[FAIL] 벽체 선이 발견되지 않았습니다.") (setq passed nil))
  )
  (if passed (princ "\n[PASS] 벽체 테스트 통과!"))
  (princ)
)

;; 2. GREEN (그리기 코드): 어설션을 통과하기 위한 최소한의 LISP 코드 구현
(progn
  (command "._layer" "_make" "A-WALL" "_color" "1" "" "")
  (command "._line" "0,0" "1000,0" "")
  (princ "\n그리기 완료.")
  (princ)
)

;; 3. REFACTOR (LISP 최적화): 코드 품질을 향상시키며 어설션이 계속 GREEN을 유지하는지 검증
(progn
  (defun draw-wall (p1 p2 / acadObj activeDoc modelSpace lineObj)
    (vl-load-com)
    (setq acadObj (vlax-get-acad-object)
          activeDoc (vla-get-ActiveDocument acadObj)
          modelSpace (vla-get-ModelSpace activeDoc)
    )
    (vla-put-ActiveLayer activeDoc (vla-item (vla-get-Layers activeDoc) "A-WALL"))
    ;; 성능과 안정성을 위해 ActiveX를 사용하여 선을 그림
    (setq lineObj (vla-AddLine modelSpace (vlax-3d-point p1) (vlax-3d-point p2)))
    (vlax-release-object lineObj)
    (vlax-release-object modelSpace)
    (vlax-release-object activeDoc)
    (vlax-release-object acadObj)
  )
  (draw-wall '(0 0 0) '(1000 0 0))
  (princ)
)
```

---

## CAD 데이터베이스 어설션

CAD 도면 상태를 검사하기 위해 다음 패턴을 활용하십시오.

### 1. 심볼 테이블 검증 (Symbol Tables Verification)
레이어, 블록 정의, 텍스트 스타일 또는 선종류가 도면에 존재하는지 확인합니다.

```lisp
;; 레이어 존재 여부 어설션
(if (tblsearch "LAYER" "A-DOOR")
    (princ "\n[PASS] A-DOOR 레이어가 존재합니다.")
    (princ "\n[FAIL] A-DOOR 레이어를 찾을 수 없습니다.")
)

;; 블록 정의 존재 여부 어설션
(if (tblsearch "BLOCK" "WINDOW-DOUBLE")
    (princ "\n[PASS] WINDOW-DOUBLE 블록 정의가 존재합니다.")
    (princ "\n[FAIL] WINDOW-DOUBLE 블록 정의를 찾을 수 없습니다.")
)
```

### 2. 선택 세트 및 엔티티 개수 검증 (Selection Set & Entity Counts)
생성된 객체들이 올바른 수량과 스펙에 맞게 그려졌는지 확인합니다.

```lisp
(setq ss (ssget "_X" '((0 . "LWPOLYLINE") (8 . "A-WALL"))))
(if (and ss (= (sslength ss) 4))
    (princ "\n[PASS] 올바른 수(4개)의 벽체 폴리라인이 생성되었습니다.")
    (princ (strcat "\n[FAIL] 예상치: 4개, 실제 검색된 수: " (if ss (itoa (sslength ss)) "0")))
)
(if ss (setq ss nil)) ; ss 해제
```

### 3. 기하학 및 속성 검사 (Geometry and Property Inspection)
DXF 리스트나 VLA 속성값을 조회하여 좌표, 길이, 색상 등의 도면 물리적 데이터를 검증합니다.

```lisp
;; DXF 조회를 통한 선의 시작점 및 끝점 좌표 검증
(setq ent (entnext))
(if ent
    (progn
      (setq dxf (entget ent))
      (setq p1 (cdr (assoc 10 dxf))
            p2 (cdr (assoc 11 dxf))
      )
      (if (and (equal p1 '(0.0 0.0 0.0) 0.001) (equal p2 '(1000.0 0.0 0.0) 0.001))
          (princ "\n[PASS] 선이 올바른 기하 좌표에 그려졌습니다.")
          (princ "\n[FAIL] 선의 좌표가 일치하지 않습니다.")
      )
    )
)
```

---

## 시스템 변수 검증

LISP 스크립트 실행 후 사용자의 CAD 작동 환경이 오염되거나 변경된 채로 방치되지 않는지 확인합니다.

```lisp
(progn
  (setq old_osmode (getvar "OSMODE"))
  ;; 도면 작업 수행
  (setvar "OSMODE" 0)
  (command "._line" "10,10" "50,50" "")
  
  ;; 정리 루틴에서 OSMODE가 원복되었는지 어설션 실행
  (setvar "OSMODE" old_osmode)
  (if (= (getvar "OSMODE") old_osmode)
      (princ "\n[PASS] OSMODE가 정상적으로 복원되었습니다.")
      (princ "\n[FAIL] OSMODE 설정이 훼손되었습니다.")
  )
  (princ)
)
```

---

## CAD Prove-It 패턴

특정 버그(예: 텍스트 회전 각도가 UCS 좌표계를 무시하는 문제, 블록이 엉뚱한 비율로 삽입되는 오류 등)를 해결할 때:

1. **재현 (RED 단계)**: 삽입된 블록의 축척 비율이나 텍스트의 실제 값을 검증하여 실패를 야기하는 어설션 셀을 작성합니다. 이를 실행하면 테스트가 실패해야 합니다.
2. **해결 (GREEN 단계)**: 시스템 변수 또는 사용자 입력을 고려하여 올바른 계산을 하도록 도면 함수를 수정합니다.
3. **검증 (GREEN 단계)**: 어설션 셀을 다시 작동하여 이제 테스트가 깨끗하게 통과하는지 확인합니다.

---

## 테스트 격리 및 정리

CAD 도면은 영구적인 상태를 유지하는 데이터베이스입니다. 각 테스트 셀이 서로 충돌하거나 도면을 더럽히는 일을 막기 위해:

- **임시 엔티티 삭제**: 테스트 종료 시 `(entdel ent)`를 호출하거나 선택 세트를 루프 돌려 임시 생성 객체를 제거합니다.
- **실행 취소(Undo) 롤백**: 테스트 드로잉 루프 전체를 `vla-StartUndoMark`와 `vla-EndUndoMark`로 감싼 후, 검증이 끝나면 `(command "._undo" "1")` 명령을 호출하여 도면을 깨끗한 원본 상태로 되돌립니다.
- **메모리 자원 해제**: 생성한 선택 세트 변수나 VLA 객체 변수들은 `(setq ss nil)` 등을 실행하여 가비지 컬렉션을 유도합니다.
