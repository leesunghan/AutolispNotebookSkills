# CAD 및 AutoLISP 관찰 가능성(Observability) 체크리스트

AutoLISP 실행 상태 추적, 성능 벤치마킹 기록, 노트북 셀 내 가독성 높은 실시간 상태 로깅을 작성하기 위한 관찰 가능성 체크리스트입니다. 이 파일은 `observability-and-instrumentation` 스킬과 함께 사용하십시오.

---

## 목차

- [진행 보고 및 콘솔 출력](#진행-보고-및-콘솔-출력)
- [실행 벤치마크 텔레메트리](#실행-벤치마크-텔레메트리)
- [에러 로깅 및 환경 진단](#에러-로깅-및-환경-진단)
- [노트북 셀 내 검증 로그 패턴](#노트북-셀-내-검증-로그-패턴)

---

## 진행 보고 및 콘솔 출력

긴 그리기 연산이 실행되는 동안 캐드 화면이 얼어붙은 것처럼 보이는 현상을 방지하고, 지나치게 불필요한 출력으로 콘솔창이 도배되지 않도록 제어합니다.

- [ ] **명령행 로깅 억제**: LISP 실행 중 내부 시스템 프롬프트가 과도하게 노출되지 않도록 `(setvar "CMDECHO" 0)`을 설정하여 로깅을 정리합니다.
- [ ] **구조화된 진행 메시지**: 프로그램의 주요 단계마다 명확한 단계별 정보를 로깅합니다 (예: `(princ "\n[INFO] 벽체 생성 초기화 중...")`).
- [ ] **사용자 실시간 진행 알림**: 반복문 내에서 대량의 객체를 그릴 때 `(prompt "창문 그리는 중...")` 또는 캐드 하단 상태 표시줄의 `(grtext -1 "작업 중... 잠시만 기다려 주십시오")` 등을 호출하여 사용자에게 진행 상황을 지속적으로 인지시킵니다.
- [ ] **NOMUTT 상태 제어**: 다량의 엔티티를 화면에 연속하여 그릴 때는 `(setvar "NOMUTT" 1)`을 켜 두지만, 사용자로부터 좌표 입력이나 텍스트 지정을 직접 받아야 할 때는 다시 원래 값인 `0`으로 꺼 주십시오.

---

## 실행 벤치마크 텔레메트리

코드 병목 지점을 찾고 기능 추가로 인한 성능 회귀(Regression)를 조기에 파악하기 위해 작동 시간을 체크합니다.

- [ ] **실행 벤치마킹**: 긴 처리를 수반하는 제도 작업의 경우 `MILLISECS` 시스템 변수(BricsCAD 및 최신 AutoCAD 지원) 또는 `DATE` 변수를 호출하여 경과 시간을 계산하고 출력합니다.

```lisp
(defun time-my-draw (/ start_time end_time elapsed_time)
  (setq start_time (getvar "MILLISECS"))
  
  ;; 실행 로직이 오는 위치...
  
  (setq end_time (getvar "MILLISECS"))
  (setq elapsed_time (/ (- end_time start_time) 1000.0))
  (princ (strcat "\n[PERF] 벽체 중심선이 " (rtos elapsed_time 2 3) "초 만에 완성되었습니다."))
  (princ)
)
```

---

## 에러 로깅 및 환경 진단

예기치 못한 프로그램 에러가 났을 때 원인을 쉽게 짚을 수 있도록 돕는 진단 환경을 구축합니다.

- [ ] **에러 시점의 환경 진단 기록**: 커스텀 `*error*` 핸들러가 오류 시점의 주요 시스템 변수 값, 오류 메시지 문자열 및 디버깅 데이터를 분석할 수 있도록 설계하십시오.
- [ ] **LISP 스택 추적**: 에러 진단을 위해 디버그용 콘솔이나 BricsCAD LISP Developer console을 켜서 예외 추적 지점을 파악하십시오.

```lisp
(setq *error* (lambda (msg)
                (princ (strcat "\n[ERROR] 처리되지 않은 오류가 발생했습니다: " msg))
                (princ (strcat "\n[DIAG] 실패 시점의 OSMODE: " (itoa (getvar "OSMODE"))))
                (princ (strcat "\n[DIAG] 실패 시점의 레이어: " (getvar "CLAYER")))
                (princ)
              )
)
```

---

## 노트북 셀 내 검증 로그 패턴

AutoLISP 노트북 셀을 구동할 때, 테스트 엔지니어 에이전트와 사용자가 검증 결과를 손쉽게 해석할 수 있도록 결과를 정형화하여 출력합니다.

- [ ] **표준화된 검증 헤더**: 검사 항목마다 성공은 `[PASS]`, 실패는 `[FAIL]` 플래그를 붙여 보기 쉽게 출력합니다.
- [ ] **요약 리포트**: 전체 검증이 끝난 후 총 테스트 개수 대비 통과 개수를 출력합니다.

```lisp
;; 표준적인 검증 어설션 출력 예시:
(progn
  (setq total_tests 2 passed_tests 0)
  (if (tblsearch "LAYER" "A-WALL")
      (progn (princ "\n[PASS] A-WALL 레이어가 생성되었습니다.") (setq passed_tests (1+ passed_tests)))
      (princ "\n[FAIL] A-WALL 레이어를 찾을 수 없습니다.")
  )
  (if (tblsearch "BLOCK" "DOOR")
      (progn (princ "\n[PASS] DOOR 블록 정의가 로드되었습니다.") (setq passed_tests (1+ passed_tests)))
      (princ "\n[FAIL] DOOR 블록 정의가 도면에 없습니다.")
  )
  (princ (strcat "\n[SUMMARY] 총 " (itoa passed_tests) "/" (itoa total_tests) "개의 테스트가 통과되었습니다."))
  (princ)
)
```
