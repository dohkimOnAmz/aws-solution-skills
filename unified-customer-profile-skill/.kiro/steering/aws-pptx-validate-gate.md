---
inclusion: fileMatch
fileMatchPattern: ".pptx-generator/**/slide-*.json"
---

# aws-pptx-generator 검증 게이트

`slide-NN.json` 작업에서 validate.py 결과를 반드시 확인하고 CRITICAL을 즉시 수정한다.

## 검증 주체

**파일 쓰기 훅은 없다.** fsWrite + fsAppend 분할 작성 과정에서 미완성 파일에 대해 훅이 돌면 CRITICAL 거짓 양성이 나고 에이전트 컨텍스트가 오염된다. 이 위험 때문에 파일 쓰기 훅은 의도적으로 폐기됐다.

대신 다음이 검증을 수행한다.

| 주체                                  | 시점                                   | 대상                                           |
| ------------------------------------- | -------------------------------------- | ---------------------------------------------- |
| **pptx-slide-builder (서브에이전트)** | 자신이 쓴 slide-NN.json 작성 완료 직후 | 단일 파일 `validate.py <파일 경로>`            |
| **메인 오케스트레이터**               | Phase 5 배치 완료 후, Phase 6 진입 전  | 디렉토리 `validate.py .pptx-generator/slides/` |

## 규칙

1. **서브에이전트 자체 검증**: builder는 `pptx-slide-builder.md` BLOCKING 규칙 4번("완료 성공 기준")에 따라 작성 종료 후 단일 파일 validate를 실행하고 CRITICAL 0건까지 수정한다. 상한 5회. 미해소 항목은 `.tmp/phase5-slides-*-result.md`에 "미해소 CRITICAL"로 기록
2. **메인 오케스트레이터 최종 검증**: 배치 완료 후 서브에이전트 결과 파일을 `readFile`로 읽어 미해소 CRITICAL 여부 확인 → 있으면 메인이 직접 수정하거나 재배치
3. **Phase 6 진입 조건**: 모든 slide-NN.json에 대해 `validate.py .pptx-generator/slides/` CRITICAL 0건
4. **재검증 루프 상한**: 동일 슬라이드에 대해 메인+서브 합쳐 총 5회 CRITICAL 재발 시 근본 원인 분석 (토큰 체계 오해, 레이아웃 선택 오류, 서브에이전트 프롬프트 편향 주입 등)

## CRITICAL 유형과 대응

| CRITICAL 유형                  | 대응                                                                      |
| ------------------------------ | ------------------------------------------------------------------------- |
| raw hex 색상 사용              | `token-atoms.md`에서 해당 색상 토큰 찾아 교체                             |
| raw pt 숫자 좌표               | `fn-mapping.md`의 그리드 토큰 규칙 재확인                                 |
| 존재하지 않는 토큰             | `tokens/*.yaml`에서 실제 키 확인 후 교체                                  |
| add_textbox/add_shape에 h 누락 | `fn-mapping.md`의 h 결정 절차로 계산하여 명시                             |
| z-order 규칙 위반              | elements 배열 순서 재배치 (배경 → 보더 → 도형 → 커넥터 → 아이콘 → 텍스트) |
| Action Title 한도 초과         | 명사구 축약 (핵심 명사만 남기고 부연 제거)                                |

## WARN 처리

WARN은 차단 사유가 아니지만 기록한다. 스트레스 테스트 모드에서는 WARN도 `issues/issue-NNN.md`에 기록하고 근본 원인을 분석한다.

## 금지 행동

- 서브에이전트가 자체 검증을 건너뛰고 작업 완료 선언
- 메인 오케스트레이터가 서브에이전트 결과 파일을 읽지 않고 요약만 신뢰
- CRITICAL 수정 없이 Phase 6 진입
- `validate.py` 없이 `build.py`를 먼저 실행
- 디렉토리 전체 validate를 서브에이전트에서 실행 (병렬 서브에이전트의 미완성 파일 간섭)
