# LAB 2 레지스터 실험 전 레포트

## 1. 제어 입력별 다음 상태 표 및 동시 동작의 계산 과정
| load | transfer | rst | 다음 stored 상태 | 다음 value 상태 |
| :---: | :---: | :---: | :--- | :--- |
| 무관 | 무관 | 1 | 0 | 0 |
| 0 | 0 | 0 | 유지 | 유지 |
| 1 | 0 | 0 | `data_in` | 유지 |
| 0 | 1 | 0 | 유지 | 이전 `stored` |
| 1 | 1 | 0 | `data_in` | 이전 `stored` |

* **동시 동작 계산 과정:** `register_pair.v`에서는 갱신 시 Non-blocking 대입 연산자(`<=`)를 사용하고 `load`와 `transfer`를 각각 독립적인 `if`문으로 검사합니다. 따라서 `load=1`, `transfer=1`이 동시에 입력될 경우, `stored`에는 현재 입력인 `data_in`이 저장되지만, 같은 클록 에지에서 `value`에는 갱신되기 직전 상태의 `stored` 값이 전달됩니다. 즉, 위아래 순서로 값이 즉시 전달되지 않고 한 클록이 더 지나야만 새로운 값이 `value`까지 넘어갑니다.

## 2. 파일별 역할 설명
* **RTL (`register_pair.v`, `input_frontend.v`, `lab2_register.v`):** 외부 스위치 및 버튼 입력을 주 클록에 동기화(디바운싱)하고, 이를 바탕으로 저장용 레지스터(`stored`)와 전달용 레지스터(`value`) 회로를 구현하여 최상위 모듈로 통합 연결하는 논리 설계 파일입니다.
* **TB (`tb_register_pair.sv`):** 작성한 레지스터 논리가 의도대로 동작하는지 확인하기 위해 인위적인 클록과 신호를 인가하고 진리표에 맞게 상태가 변하는지 스스로 검사(Self-Checking)하는 테스트벤치 파일입니다.
* **`simulation.json`:** VS Code 환경과 Icarus 시뮬레이터가 컴파일해야 할 소스 파일 목록, 시뮬레이션 탑(Top) 모듈의 이름 등 시뮬레이션 구동 환경 설정을 명시하는 파일입니다.
* **XDC (`lab2_register.xdc`):** Vivado 합성 및 구현 단계에서 사용되며, Combo II 보드의 스위치, 버튼, 클록(1 kHz), LED의 물리적 핀 매핑과 I/O 표준(LVCMOS33)을 설정하는 물리 제약조건 파일입니다.

## 3. 정상/변경/원복 실행 로그 및 검사 시각 비교
`value <= stored` 코드를 `value <= data_in`으로 수정하여 발생한 오류를 정상 및 복구 로그와 비교하였습니다.
* **[정상 시뮬레이션 로그](../../evidence/03/vscode/simulation.txt):** `LAB2_PASS register_pair checks=7`, 종료 시각: 66 ns 통과.
* **코드 변경 후 실패 로그:** `LAB2_FAIL transfer stored not live input time=26000`
  * **해석:** 26 ns 시점에서 `transfer=1`이 되어 이전 `stored` 값(A)이 전달되어 기댓값 `8'haa`가 나와야 하지만, 코드를 수정하여 입력값(3)이 직접 전달되는 바람에 불일치가 감지되어 오류($fatal) 처리 및 중단되었습니다.
* **원복 시뮬레이션 로그:** `LAB2_PASS register_pair checks=7`, 종료 시각: 66 ns 통과로 다시 정상 작동을 확인했습니다.

## 4. 파형 분석
> **[ VaporView 전체 파형 ](../../evidence/03/vscode/wave.png)**
* **25 ns 분석:** `data_in`에 3이 인가되어 있으나 `load=0`, `transfer=1`이므로 입력은 무시되고 `value`에는 15 ns에 이미 저장해 두었던 값 `A`가 정상적으로 전달되었습니다.
* **35 ns 분석 (동시 실행):** `load=1`과 `transfer=1`이 동시에 발생했습니다. 이때 `stored`는 새로운 `data_in`인 3으로 갱신되나, Non-blocking 특성 때문에 `value`는 `stored`가 3으로 바뀌기 전의 기존 값인 `A`를 계속 유지하는 것을 확인할 수 있습니다.
* **45 ns 분석:** 계속해서 제어 신호가 1인 상태입니다. 이전 클록에서 갱신되었던 `stored` 값 3이 비로소 이번 상승 에지에서 `value`로 전달되어 양쪽 레지스터 모두 값이 3이 되는 것을 볼 수 있습니다. 

## 5. 보드 입력 순서와 예상 LED 계산
실제 보드에서 실행할 때의 순서와 예상 LED 값입니다. (DIP 스위치 세팅 후 K4 버튼을 먼저 눌러 리셋합니다).
* **sw=A1 설정 후 N8 실행:** (data_in=A, load=1, transfer=0) -> `stored`에 A가 저장되고 `value`는 0 유지. **예상 LED: `A0`**
* **sw=32 설정 후 N8 실행:** (data_in=3, load=0, transfer=1) -> `value`에 A 전달. `stored`는 A 유지. **예상 LED: `AA`**
* **sw=33 설정 후 N8 실행:** (data_in=3, load=1, transfer=1) -> 동시 동작 발생. `stored`에 3 저장, `value`는 이전 `stored` 값 A 유지. **예상 LED: `3A`**
* **sw=33 상태에서 N8 다시 실행:** `stored`에 3 재저장, `value`에는 이전 값 3 전달. **예상 LED: `33`**
* **sw=F0 설정 후 N8 실행:** (data_in=F, load=0, transfer=0) -> 입력 무시. 둘 다 이전 상태 유지. **예상 LED: `33`**
* **K4 버튼 누름:** 모든 레지스터 비동기(즉각) 리셋. **예상 LED: `00`**