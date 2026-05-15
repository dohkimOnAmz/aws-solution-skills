---
inclusion: always
---

# aws-pptx-generator 파이프라인 순서 강제

스킬 트리거 시 파이프라인 순서(Phase 0 초기 셋업 → Phase 1 → ... → Phase 7 시각 분석)를 건너뛰거나 역행하지 않는다. 진입점은 Phase 0 초기 셋업이며, 본 파이프라인은 Phase 1~7의 **7단계**로 구성된다(SKILL.md와 동일 기준).

## 선행 아티팩트 체크리스트

다음 Phase에 진입하기 전 선행 아티팩트 존재를 반드시 확인한다.

| 진입 Phase | 선행 아티팩트                                                                               |
| ---------- | ------------------------------------------------------------------------------------------- |
| Phase 1    | `check_env.py --install`이 성공 종료했음                                                    |
| Phase 2    | `.pptx-generator/analysis.md` + `.pptx-generator/questions.md` 존재, 사용자 답변 반영 완료  |
| Phase 3    | `.pptx-generator/scenario.md` 존재, 사용자 확정 완료 (스트레스 테스트 모드는 자동 확정)     |
| Phase 4    | `.pptx-generator/slides/slide-NN.md` (Cover/Agenda/Section/ThankYou) 존재                   |
| Phase 5    | `.pptx-generator/slides/slide-NN.md` (Content) 존재, validate_md.py CRITICAL 0건            |
| Phase 6    | `.pptx-generator/slides/slide-NN.json` 전체 존재, validate.py CRITICAL 0건, SVG 임베딩 완료 |

선행 아티팩트가 누락되면 **이전 Phase로 복귀**한다. 사용자에게 "건너뛰어도 될까요?"를 묻지 않는다. 파이프라인은 결정론적이다.

## GATE 게이트 규칙

일반 모드:

- GATE_1(Phase 1 종료), GATE_2(Phase 2 종료), GATE_3(Phase 3 종료), GATE_4(Phase 5 종료)에서 사용자 확정을 받는다
- 사용자 응답 없이 다음 Phase로 진행 금지

스트레스 테스트 모드 (`references/stress-test-scenario.md` 트리거):

- 모든 GATE를 자동 통과로 처리한다
- 사용자의 명시적 중단 요청("중단해", "stop")이 오면 현재 단계 완료 후 중단

## 금지 행동

- 사용자가 "바로 pptx 만들어줘"라고 해도 Phase 1~2를 건너뛰고 Phase 5로 점프하지 않는다
- Phase 3 배치 처리 중간에 Phase 5 작업을 섞지 않는다
- 한 Run의 Phase 6 빌드가 실패했을 때 Phase 2 시나리오를 임의로 수정하지 않는다 (근본 원인 분석 후 복귀)

## Plan mode 권고

Phase 3·5의 배치 분할(어느 슬라이드를 어느 서브에이전트에 묶을지)은 서브에이전트 호출 **전** plan mode(Shift+Tab×2) 또는 `/ultraplan`으로 확정한다. 분할 drift는 200줄 결과 파일 diff보다 10줄 plan에서 잡는 게 훨씬 싸다. 스트레스 테스트 모드도 동일하게 적용 — 모든 GATE가 자동 통과여도 배치 분할 plan은 유지한다.

## 예외

사용자가 명시적으로 "Phase X만 다시 실행해줘"라고 요청하면 해당 Phase의 선행 아티팩트가 존재하는지 확인한 뒤 단독 실행한다. 선행 아티팩트가 없으면 요청을 거절하고 어디서부터 시작해야 하는지 안내한다.
