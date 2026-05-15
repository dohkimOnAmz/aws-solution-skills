---
inclusion: manual
---

# aws-pptx-generator 스트레스 테스트 모드

`references/stress-test-scenario.md`에 정의된 자율 테스트 시나리오를 실행하는 모드.

이 스티어링은 **manual inclusion**이다. 사용자가 `#aws-pptx-stress-test-mode` 컨텍스트 키를 명시하거나 아래 트리거 키워드를 언급할 때만 로드된다.

## 진입 트리거

- "pptx-generator 스트레스 테스트"
- "aws-pptx-generator 전방위 테스트"
- "pptx 스킬 스트레스 테스트 해줘"
- "슬라이드 생성 스트레스 테스트"
- "pptx-generator QA 시나리오"

## 모드 활성 시 행동 변화

1. **문서 진입**: `.kiro/skills/aws-pptx-generator/references/stress-test-scenario.md`를 먼저 읽고 절차를 따른다
2. **GATE 자동 통과**: 일반 모드의 GATE_1~GATE_4를 모두 자동 통과로 처리한다 (`aws-pptx-pipeline-order.md`의 GATE 규칙 예외)
3. **작업 디렉토리 고정**: `.pptx-generator/stress-test/`를 작업 디렉토리로 사용. 일반 `.pptx-generator/slides/`를 건드리지 않는다
4. **질문 자동 답변**: questions.md의 질문을 scenario.md "테스트 덱 성격" 표의 값(청중·시간·페이지번호 등)으로 자동 답변한다
5. **엣지케이스 의도적 주입**: scenario.md 설계 단계에서 엣지케이스 목록(E01~E29, 추가 시 갱신)을 자연스럽게 녹여 넣는다. 서브에이전트 프롬프트에 직접 주입하지 않는다
6. **이슈 기록 필수**: 발견된 모든 이슈를 `.pptx-generator/stress-test/issues/issue-NNN.md`에 기록. 근본 원인 분석 포함
7. **중단 조건**: 사용자의 명시적 중단 요청("중단해", "stop") 시 현재 단계 완료 후 중단. 그 외에는 final-report.md까지 자율 실행
8. **재활용 금지**: 이전 스트레스 테스트의 slide md를 절대 재활용하지 않는다. Step 0에서 `stress-test/` 전체 삭제 후 백지 시작

## 테스트 덱 구조

**단일 덱 20장 (본문 16장)** 을 처음부터 끝까지 실행한다. 이전 버전의 Run A/B/C 난이도 분할 구조는 폐기됐다.

- Cover 1 + Agenda 1 + Section 1 + Content 16 + ThankYou 1 = 20장
- 슬라이드별 주제는 단일 서사로 엮지 않고 AWS 전반 영역을 교차 배치하여 엣지케이스·레이아웃·밀도 패턴을 최대로 압축한다
- 상세 목차와 엣지케이스 주입 매트릭스는 `references/stress-test-scenario.md`에 정의되어 있다

## 스킬 자체 수정 허용

스트레스 테스트에서 발견한 근본 원인이 스킬 자체에 있으면 수정할 수 있다. 단, 수정 내역을 반드시 `.pptx-generator/stress-test/fixes-applied.md`에 기록한다.

수정 대상이 될 수 있는 파일:

- `.kiro/skills/aws-pptx-generator/SKILL.md`
- `.kiro/skills/aws-pptx-generator/references/*.md`
- `.kiro/skills/aws-pptx-generator/scripts/*.py`
- `.kiro/skills/aws-pptx-generator/subagents/*.md`
- `.kiro/skills/aws-pptx-generator/tokens/*.yaml`

## 금지 행동

- 일반 사용자 세션에서 트리거 없이 이 모드 활성화
- `.pptx-generator/slides/` 일반 경로에 스트레스 테스트 산출물 쓰기
- GATE 자동 통과 규칙을 일반 모드에도 적용
- 서브에이전트에 엣지케이스 주입 지시 직접 전달 (편향 금지)

## 시각 분석 판정 신뢰 규칙 (중요)

스트레스 테스트에서 시각 분석 에이전트(`pptx-visual-analyzer`)의 판정이 오판정일 수 있다는 사실이 이전 실행에서 드러났다(보더 색 혼동으로 "VPC 외부 이탈"이라는 잘못된 회귀 판정). 메인 오케스트레이터는 이를 방지하기 위해 다음 규칙을 반드시 적용한다.

### 1. 심각도 상향 교차 검증 (BLOCKING)

시각 분석이 **동일 슬라이드의 동일 이슈를 연속 두 iter 사이에 심각도 상향**(예: Medium → Critical, P2 → P1, "정상" → "회귀")하는 판정을 내리면, 메인은 **수정 작업에 착수하기 전에** 아래 순서를 수행한다.

1. 해당 슬라이드의 렌더 PNG를 `readFile`로 직접 열어본다
2. 시각 분석 에이전트가 기술한 증거(좌표·픽셀 거리·색 매핑)를 이미지에서 직접 확인한다
3. 특히 **근접한 다중 보더·선이 있는 영역**에서는 보더 색 구분이 에이전트 주장과 일치하는지 확인한다
4. 에이전트 주장이 이미지에서 **확인되지 않으면** 해당 iter 판정을 "신뢰도 낮음"으로 표시하고, 해당 슬라이드에 대해 `scope=focus` 재분석을 요청한다
5. 교차 검증이 일치하면 수정에 착수한다

### 2. 증거 결여 판정은 수정 거부

시각 분석 보고서에서 **자연어만으로 기술된 위상 판정**("바깥", "외부", "이탈", "침범" 등)에 좌표·픽셀 거리가 동반되지 않으면, 메인은 이를 **수정 대상에서 제외**하고 "증거 불충분 — 재분석 요구"로 이슈 파일에 기록한다. BLOCKING 3 위반이기 때문이다.

### 3. 회귀 승격의 3요소 확인

시각 분석이 회귀 판정을 내릴 때는 다음 3요소가 보고서에 모두 있어야 한다 (visual-analyzer BLOCKING 9).

- 이전 iter 해당 이슈 기술 **원문 인용**
- 현재 iter **측정값** (좌표 또는 픽셀 거리)
- **수량화 비교** (예: "이전 12px 이격 → 현재 -5px 겹침")

하나라도 결여되면 메인은 그 승격을 수용하지 않고 "잔존 동일 심각도"로 취급한다.

### 4. 매니페스트 검증 우회 금지

`render_to_png.py`는 출력 디렉토리에 `.render-manifest`를 기록한다. 시각 분석을 호출하기 전 메인은 PPTX 빌드 후 반드시 재렌더를 실행하여 매니페스트를 갱신한다. **매니페스트의 pptx_mtime과 실제 PPTX mtime이 어긋난 상태로 시각 분석을 호출하지 않는다.**

### 5. 판정 철회 기록 의무

이전 iter에서 특정 판정에 근거해 수정한 내용이 **후속 iter의 재측정으로 오판정임이 밝혀지면**, 다음을 즉시 기록한다.

- 해당 `issue-NNN.md`에 "## 판정 이력" 섹션 추가 + "iter N 판정 → iter M 픽셀 측정으로 철회" 명시
- `fixes-applied.md`에 해당 수정을 "rollback: 오판정 근거"로 주석 표시
- 수정이 결과적으로 개선이었다 하더라도 **근거가 오판정이었다는 사실**은 반드시 기록 (향후 신뢰도 조정 용도)

### 6. 금지 행동

- 시각 분석의 "회귀" 또는 "P1 승격" 판정을 교차 검증 없이 즉시 수정 대상으로 처리
- 자연어만 있고 좌표 없는 증거를 근거로 수정 시작
- 같은 슬라이드에 대해 iter마다 상반된 판정이 나오는데 최신 판정만 믿기 — 이럴 때는 두 판정 모두 증거 기반으로 재검증
