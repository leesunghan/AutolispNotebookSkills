---
name: debugging-and-error-recovery
description: AutoLISP Notebook의 코드 실행 오동작, 기하 연산 좌표 어긋남, 캐드 프리징 등의 에러를 체계적으로 추적하고 복구합니다.
---

# 디버깅 및 에러 복구 (AutoLISP Notebook 에디션)

## 개요

CAD 자동화 프로세스 개선을 위한 체계적인 디버깅 방식입니다. AutoLISP 코드 실행 중 오류가 나거나 계산된 좌표 위치에 엉뚱하게 객체가 얹어지는 등의 비상 오작동이 터졌을 때는 개발을 즉시 중단하고, 캐드 명령 이력을 수집하여 문제점의 직접적인 원인을 찾아 복구 조치해야 합니다.

## 비상 상황 전면 중단 규칙 (The Stop-the-Line Rule)

AutoLISP 셀 구동 중 예기치 못한 실패가 나타나면:
1. **즉각** 다른 후속 셀들의 구동 및 타이핑을 **중단하십시오(STOP).**
2. 발생 당시의 **도면 정보와 로그를 수집하십시오** (콘솔에 출력된 AutoLISP 에러 문구 캡처, 찌그러진 선들의 실제 기하 좌표 수치 기록).
3. **AutoLISP 분석 진단 절차**를 밟아 문제를 진단하십시오.
4. **근본적 원인을 패치하십시오** (예: 잘못된 법선 벡터 수식 보정, 변수 `nil` 값 비교 로직 보완).
5. **어설션 검증**을 통해 버그 재발 방지 대책을 마련하십시오.
6. **CAD 시스템 변수**(`OSMODE`, `CMDECHO` 등)가 정상으로 되돌아온 것을 확인하고 작업을 재개하십시오.

---

## AutoLISP 분석 진단 체크리스트 (The AutoLISP Triage Checklist)

### Step 1: 재현 및 캐드 콘솔창 확인 (Reproduce & Capture)
문제가 나타난 코드 셀을 실행하고 즉시 AutoCAD/BricsCAD 창에서 **`F2`** 단추를 눌러 **전체 명령 히스토리 창**을 띄우십시오. 콘솔에 박힌 에러 메시지를 수집합니다.

자주 나타나는 AutoLISP 예외 에러 메시지 해석:
- `bad argument type: lentityp nil`: 엔티티 요소를 가리켜야 하는 API(`entget`, `vlax-ename->vla-object`)에 `nil` 값이 들어왔습니다. 사용자가 객체 지정 도중 빈 공간을 클릭했거나 중도 취소했는데, 리턴값 존재 여부를 대조하는 방어막(`if`)을 두지 않아 발생합니다.
- `bad argument type: numberp: nil`: 곱하기나 더하기 등 사칙연산 함수에 숫자가 아닌 `nil` 변수가 인입되었습니다. 기하 투영 거리나 중간 연산 좌표가 잘못 도출되었음을 시사합니다.
- `too few arguments` 또는 `too many arguments`: 사용자 정의 AutoLISP 함수 호출 시 전달한 인자의 개수와 함수의 매개변수 선언 개수가 불일치합니다.
- `malformed list on input` 또는 `extra right paren`: 소스 코드 내에서 열린 괄호(`(`)와 닫힌 괄호(`)`)의 총개수 균형이 깨져 인터프리터가 읽지 못하는 상태입니다.

### Step 2: 버그 위치 특정 (Localize)
오동작이 유발되는 AutoLISP 소스의 특정 라인을 포착하십시오:
- **괄호 짝 점검**: 코드를 VS Code 에디터로 옮겨 괄호 짝을 맞춰주는 자동 매칭 도구를 활용해 균형을 확인합니다.
- **선택 세트 경계**: 루프 순회 인덱스가 실제 크기를 초과하여 참조하고 있지 않은지 살핍니다.
- **ActiveX 경계**: 선이나 폴리선이 아닌 문장(Text) 객체에 대고 `vla-Offset` 등을 적용해 캐드 엔진 내부 예외를 유발하는지 조사합니다.

### Step 3: 디버깅용 코드 간소화 (Reduce)
원인을 좁히기 위해:
- 문제가 의심되는 기하 벡터 수학 공식이나 엔티티 질의문만 떼어냅니다.
- CAD 명령줄(Command Line)에 직접 개별 한 행씩 쳐 넣으면서 수동 연산을 수행해 봅니다.
- 중간 변수들의 상태를 확인하기 위해 `(princ var1)`을 코드 곳곳에 임시로 얹어, 어느 행을 넘어갈 때 데이터가 `nil`로 유실되는지 추적합니다.

### Step 4: 근본 원인 해결 (Fix)
방어 코드를 엄격하게 구축하십시오:
- 객체 존재 검증: `(if (and ent (setq dxf (entget ent))) ...)`
- 개체 레이어 및 형식 판별: `(if (= (cdr (assoc 0 dxf)) "LINE") ...)`
- 분모 0 에러 예방: `(if (not (equal dist 0.0 1e-6)) ...)`

### Step 5: 비상 상황 복구 처리 (Guard and Restore)
코드 셀 도중 중단 사태가 터지더라도 환경 변수가 안전 복원되도록 로컬 에러 수집기를 구성하십시오:
```lisp
(defun *error* (msg)
  (setvar "OSMODE" old-osmode)
  (setvar "CMDECHO" old-cmdecho)
  (command "._undo" "_end")
  (setq *error* old-error)
  (princ (strcat "\n에러 핸들러 복구 완료: " msg))
  (princ)
)
```

---

## 대표적인 AutoLISP 에러 예방 가이드

| 캐드 비상 현상 | 유발 요인 | 해결 방안 및 예방 가이드 |
|-----|-------|--------------------|
| **도면 선이 엉뚱한 점을 향해 찌그러짐** | 드로잉 명령 실행 중에 사용자가 켜 둔 OSNAP의 간섭을 받음 | 작도 함수 진입 직전 `(setvar "OSMODE" 0)`으로 비활성화 처리하고, 작도 완료 후 복원합니다. |
| **타 기종/해외 캐드에서 작동 안 됨** | 영문 전용 접두사 및 기본 명령 마킹 누락 | 호출하는 모든 명령에 다국어 접두사 `._`를 붙입니다. (예: `(command "._offset" ...)`). |
| **드로잉 도중 취소(Cancel) 시 다운됨** | 사용자 입력 대기 함수가 취소되어 `nil`을 반환한 상태의 예외 처리 누락 | 분기 처리로 감싸줍니다: `(if (setq ent (entsel "\n벽선을 선택하십시오: ")) (progn ...))` |
| **ActiveX 오프셋 연산 실행 실패** | 오프셋 타겟 형상의 굴곡이 반경보다 작아 겹침/파손 예외 발생 | ActiveX 예외를 우회할 수 있도록 `vl-catch-all-apply` 구문으로 호출을 제어합니다. |

```lisp
;; 안전한 ActiveX 오프셋 제어 예시
(setq result (vl-catch-all-apply 'vla-Offset (list vla-obj offset-dist)))
(if (vl-catch-all-error-p result)
    (princ "\n기하 구조적 한계로 오프셋 생성이 불가능합니다.")
    (princ "\n오프셋 처리가 성공적으로 실행되었습니다.")
)
```
