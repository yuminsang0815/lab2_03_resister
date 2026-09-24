# 실험 후 레포트: 레지스터

작성일 2026-09-24.

[실험 전 레포트](../pre/03_register.md) · [해시·입력 기록](../../build/sim/result.json)

## Vivado GUI 과정과 사전 결과 비교

v2.0.1 템플릿 환경에서 Vivado 2026.1 GUI의 New Project를 실행하여 `lab2_register` 프로젝트를 생성했습니다. 타깃 부품은 `xc7s75fgga484-1` (Spartan-7, 패키지 fgga484, 속도 등급 -1)입니다.

RTL 모듈인 `src/register_pair.v`, `src/input_frontend.v`, `src/lab2_register.v`를 Design Sources로 등록하고, 자체 검증용 테스트벤치 `sim/tb_register_pair.sv`를 Simulation Sources에, 핀 및 클록 제약 파일 `constraints/lab2_register.xdc`를 Constraints에 추가했습니다. 이때 Copy sources into project 옵션을 해제하여 VS Code 작업 폴더의 원본 소스를 직접 참조하도록 구성했습니다.

설계 최상위 모듈(Design Top)은 `lab2_register`, 시뮬레이션 최상위 모듈(Simulation Top)은 `tb_register_pair`로 분리 지정했습니다.

Run Simulation → Run Behavioral Simulation을 실행하여 [실제 GUI 시뮬레이션 로그](../../evidence/03/vivado/simulation.log)에서 `LAB2_PASS register_pair checks=7` 출력과 66 ns($finish called at 66000 ps) 정상 종료를 확인했습니다. VS Code(Icarus Verilog/VaporView)의 사전 시뮬레이션 결과와 비교했을 때, 15 ns의 load 독립 동작, 25 ns의 live input이 아닌 직전 stored 전달, 35 ns의 동시 실행 시 이전 값 유지(nonblocking 대입 특성), 45 ns의 갱신값 전달, 55 ns 유지, 65 ns 동기 리셋 우선순위의 7가지 검사 시점과 16진수 출력값({stored, value})이 완전히 일치함을 대조했습니다.

## 합성·구현·bit

Run Synthesis → Run Implementation → Generate Bitstream을 GUI에서 단계별로 실행하여 완료했습니다. [GUI 빌드 로그](../../evidence/03/vivado/build.log)를 보관했습니다.

* **생성 파일**: `vivado/lab2_register.runs/impl_1/lab2_register.bit`
* **배포 파일**: [lab2_register.bit](../../evidence/03/vivado/lab2_register.bit) (SHA-256 해시값 기록 완료)
* **핀 배치 확인**: Elaborated Design 및 Implemented Design의 I/O Ports 창에서 주 클록(`clk`=B6), 리셋(`rst`=K4), 버튼(`button`=N8), 8비트 스위치(`sw[7:0]`), 8비트 LED(`led[7:0]`) 포트가 XDC에 선언된 대로 `LVCMOS33` 규격과 해당 핀 번호에 정상 매핑되었음을 확인했습니다.

### 타이밍 및 경고(Warning) 분석

1. **내부 클록 타이밍 결과**:
   * 온보드 1 kHz 클록(`trainer_1khz`, 주기 1,000,000.000 ns) 제약 조건에서 Open Implemented Design → Timing Summary를 확인한 결과, Setup WNS = 999997.812 ns, Hold WHS = 0.119 ns, Failing Endpoints = 0개(전체 72개)로 내부 동기식 경로의 타이밍을 안정적으로 만족했습니다. 보드 주기가 1 ms로 길기 때문에 WNS 여유가 크게 나타납니다.
2. **TIMING-18 경고**:
   * LED 출력 8개에 대한 외부 지연 제약이 없어 발생한 경고입니다. LED는 외부 동기 클록으로 샘플링되는 인터페이스가 아니라 시각적 관찰용 포트이므로 타이밍 제약을 지정하지 않았으며, 기능 및 하드웨어 동작에 이상이 없음을 확인했습니다.
3. **DRC 경고 (CFGBVS-1)**:
   * Bank 0의 전압 속성(CFGBVS/CONFIG_VOLTAGE)이 지정되지 않아 발생한 경고입니다. 보드 회로도의 공급 전압을 확인한 뒤 적용해야 하므로 경고 제거만을 목적으로 임의의 전압을 입력하지 않았으며, 비트스트림 생성은 성공적으로 완료되었습니다.

## 보드 기록·촬영 상태

Combo II-DLD S75 보드의 전원과 JTAG 케이블을 연결하고 온보드 메인 클록을 1 kHz로 설정했습니다. Vivado Hardware Manager의 Auto Connect를 통해 `xc7s75` 디바이스를 인식한 뒤 `lab2_register.bit`를 프로그래밍했습니다.

K4 푸시버튼으로 리셋을 인가한 뒤, DIP 스위치(`sw[7:4]`: data_in, `sw[1]`: transfer, `sw[0]`: load)를 설정하고 N8 버튼을 눌러 상태 전이를 실측했습니다.

| 순서 | 설정 스위치 sw[7:0] (hex) | 제어 신호 분석 | N8 1회 누름 후 실측 LED[7:0] (hex) | 동작 상태 및 해석 | 사진 |
|---|---|---|---|---|---|
| 1 | 초기 상태 (K4 리셋 누름) | rst=1 | 00 | 두 레지스터 모두 0으로 초기화 | [초기화](../../evidence/03/board/photos/step1_reset.jpg) |
| 2 | A1 | data=A, load=1, trans=0 | A0 | data_in(A)을 stored에 저장, value는 0 유지 | [A 저장](../../evidence/03/board/photos/step2_loadA.jpg) |
| 3 | 32 | data=3, load=0, trans=1 | AA | live input(3)이 아닌 기존 stored(A)를 value로 전달 | [A 전달](../../evidence/03/board/photos/step3_transA.jpg) |
| 4 | 33 | data=3, load=1, trans=1 | 3A | 동시 실행: stored는 3 저장, value는 직전 stored(A) 반영 | [동시 실행1](../../evidence/03/board/photos/step4_simul1.jpg) |
| 5 | 33 (다시 누름) | data=3, load=1, trans=1 | 33 | 다음 에지에서 새로 갱신된 stored(3)가 value로 전달 | [동시 실행2](../../evidence/03/board/photos/step5_simul2.jpg) |
| 6 | F0 | data=F, load=0, trans=0 | 33 (유지) | 두 제어가 0이므로 입력 F를 무시하고 기존 상태 유지 | [유지](../../evidence/03/board/photos/step6_hold.jpg) |
| 7 | K4 리셋 인가 | rst=1 | 00 | 두 제어가 켜져 있어도 리셋이 우선하여 00 초기화 | [최종 리셋](../../evidence/03/board/photos/step7_reset.jpg) |

[레지스터 보드 시연 영상](../../evidence/03/board/videos/demo.mp4)

* DIP 스위치 조작 후 N8 버튼을 누르면 `input_frontend` 모듈의 디바운스 로직(20 ms 안정화)을 거쳐 1클록 폭의 `press` 펄스가 생성되어 정확히 1회의 에지만 레지스터에 인가됨을 확인했습니다.
* `sw=33` 상태에서 N8 버튼을 두 번 연속 누르는 조작을 통해, 첫 번째 누름에서는 `3A`, 두 번째 누름에서는 `33`으로 바뀌는 nonblocking 대입 특성을 보드 LED를 통해 실측 검증했습니다.

## 결론

독립된 두 `if` 문과 nonblocking 할당(`<=`)으로 구성된 `register_pair` 코어의 7가지 전수 동작에 대해 VS Code Icarus Verilog와 Vivado GUI XSim의 시뮬레이션 결과가 100% 동일함을 확인했습니다.

입력 저장 레지스터(`stored`)와 출력 전달 레지스터(`value`)가 분리되어 있어, `load`와 `transfer`가 동시에 활성화되더라도 동일 클록 에지에서는 직전 저장값이 `value`로 전달되고 다음 에지에서 새 입력이 반영되는 파이프라인 레지스터의 기본 동작 특성을 증명했습니다.

Spartan-7(`xc7s75fgga484-1`) 타깃으로 합성, 구현, 내부 1 kHz 클록 타이밍 충족(WNS/WHS 마진 확보) 및 비트스트림 생성을 완료하였으며, Combo II-DLD S75 보드 상에서 스위치 설정 및 버튼 입력을 통해 이론적 전이 상태와 LED 점등 패턴이 정확히 일치함을 실측 검증했습니다.