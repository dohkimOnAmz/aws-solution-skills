---
inclusion: always
---

# aws-pptx-generator 서브에이전트 위임 규칙

Phase 3/5/7 및 유틸리티(감사)에서 서브에이전트를 호출할 때 편향을 주입하지 않고 contextFiles를 정확히 전달한다.

## pptx-slide-simulator (Phase 3)

**전달 가능한 내용**:

- 슬라이드 번호 (예: "slide-04, slide-05, slide-06")
- 각 슬라이드의 제목 (scenario.md에 명시된 것)
- 각 슬라이드의 **유형** (Cover / Agenda / Section / Content / ThankYou) — 서브에이전트가 template-slide.md의 해당 타입 템플릿을 선택할 수 있도록 명시
- 소스 데이터 파일 경로

**중요**: 위임 가능 대상은 **5종 전부**다. Cover/Agenda/Section/ThankYou의 md도 Phase 3에서 생성되어야 Phase 4 JSON 매핑이 성립한다. "Content만 서브에이전트로 보내고 나머지는 오케스트레이터가 직접 쓰기"는 폐기된 패턴이다. 오케스트레이터가 md를 "직접 작성"하는 경로는 파이프라인 어디에도 없다 — 모든 md는 `pptx-slide-simulator`가 쓴다.

**배치 구성 원칙**:

- 특수 4장(Cover/Agenda/Section/ThankYou)은 구조 고정이므로 **한 배치에 몰아서** 처리 가능 (예: `slide-01, slide-02, slide-03, slide-20`)
- Content는 5~7장 단위 배치
- 배치 1개에 특수+Content를 섞는 것도 허용 (예: `slide-01, slide-02, slide-03, slide-04, slide-05`)

**프롬프트 포맷 (Like this:)**:

```
slide-04, slide-05, slide-06을 시뮬레이션하라.
각 슬라이드의 타입: slide-04=Content, slide-05=Content, slide-06=Section
scenario.md: .pptx-generator/scenario.md
source: <소스 경로>
```

레이아웃·시각화 패턴·콘텐츠 구조·엣지케이스는 서브에이전트가 contextFiles의 references와 scenario.md에서 자율 판단한다. 스트레스 테스트 모드에서도 엣지케이스는 scenario.md 설계에 녹여 넣고 프롬프트에는 주입하지 않는다.

**필수 contextFiles**:

- `.kiro/skills/aws-pptx-generator/references/template-slide.md`
- `.pptx-generator/scenario.md`
- `.kiro/skills/aws-pptx-generator/references/deck-foundations.md`
- `.kiro/skills/aws-pptx-generator/references/deck-storytelling.md`
- `.kiro/skills/aws-pptx-generator/references/layout-patterns.md`
- `.kiro/skills/aws-pptx-generator/references/layout-density.md`
- `.kiro/skills/aws-pptx-generator/references/visualization-patterns.md`
- `.kiro/skills/aws-pptx-generator/references/diagram-principle.md`
- `.kiro/skills/aws-pptx-generator/references/diagram-ascii.md`
- `.kiro/skills/aws-pptx-generator/references/aws-architecture-patterns.md`
- `.kiro/skills/aws-pptx-generator/references/element-rules.md`
- 소스 데이터 파일(들)

## pptx-slide-builder (Phase 5)

**전달 가능한 내용**:

- `.pptx-generator/slides/slide-NN.md` 파일 경로 (2~3장)

**프롬프트 포맷 (Like this:)**:

```
담당 슬라이드: slide-11, slide-12, slide-14 (Content)
각 md 경로:
  - .pptx-generator/slides/slide-11.md
  - .pptx-generator/slides/slide-12.md
  - .pptx-generator/slides/slide-14.md
```

fn 선택·좌표·토큰·z-order는 서브에이전트가 contextFiles의 fn-mapping.md + element-rules.md + diagram-svg.md + svg-antipatterns.md 기반으로 자율 결정한다.

**필수 contextFiles**:

- 담당 `slide-NN.md` 파일(들)
- `.kiro/skills/aws-pptx-generator/references/deck-foundations.md`
- `.kiro/skills/aws-pptx-generator/references/token-atoms.md`
- `.kiro/skills/aws-pptx-generator/references/token-composite.md`
- `.kiro/skills/aws-pptx-generator/references/fn-mapping.md`
- `.kiro/skills/aws-pptx-generator/references/slide-types.md`
- `.kiro/skills/aws-pptx-generator/references/element-rules.md`
- `.kiro/skills/aws-pptx-generator/references/diagram-svg.md`
- `.kiro/skills/aws-pptx-generator/references/svg-antipatterns.md`
- `.kiro/skills/aws-pptx-generator/references/icon-catalog.md`
- `.kiro/skills/aws-pptx-generator/references/layout-patterns.md`
- `.kiro/skills/aws-pptx-generator/references/layout-density.md`
- `.kiro/skills/aws-pptx-generator/references/diagram-principle.md`

## 공통 호출 규칙

- **병렬 실행**: 슬라이드를 서브에이전트당 2~3장씩 분할하여 최대 5개까지 병렬 호출
- **결과 파일 반드시 읽기**: 서브에이전트 완료 후 `.tmp/phase3-slides-[범위]-result.md` 또는 `.tmp/phase5-slides-[범위]-result.md`를 `readFile`로 읽어 전체 맥락 파악. 요약 응답만으로 판단 금지
- **자가 검증 책임 분리**: 서브에이전트는 자기가 쓴 파일에 대해 단일 파일 validate를 자체 실행한다 (`validate_md.py <파일>` 또는 `validate.py <파일>`). 메인 오케스트레이터는 배치 완료 후 **디렉토리 전체** validate를 실행해 슬라이드 간 일관성(번호 gap 등)까지 확인한다. CRITICAL 0건을 다음 배치 진입 조건으로 한다.
  - 파일 쓰기 훅은 분할 쓰기(fsWrite+fsAppend)와의 충돌로 폐기됐다. 훅이 검증을 대체하지 않는다.

## pptx-skill-auditor (유틸리티 — 감사)

일반 워크플로우에서는 호출하지 않는다. 사용자가 "스킬 감사", "품질 점검", "교차검증"을 명시적으로 요청할 때만 호출한다.

**전달 가능한 내용**:

- 작업 트리거 한 줄 ("aws-pptx-generator 스킬을 감사한다")
- 사용자가 명시 지정한 매개변수만 (`scope: readonly|full|fix-only`, `loop_limit: N`, `target_root: 경로`)

**프롬프트 포맷 (Like this:)**:

```
aws-pptx-generator 스킬을 감사한다.
```

(사용자가 명시 지정한 매개변수가 있으면 추가: `scope: readonly|full|fix-only`, `loop_limit: N`, `target_root: 경로`)

감사 에이전트는 실행 시점에 직접 디렉토리를 스캔해 파일 인벤토리를 구축한다. 파일 목록 나열·평가 기준 재정의·우선순위 지시·특정 파일 결함 힌트·이전 결함 정보는 "실행 시점 스캔" 원칙을 우회시키므로 프롬프트에 넣지 않는다. 5 Phase 절차·체크리스트·BLOCKING 규칙은 전부 시스템 프롬프트에 내장되어 있어 재기술하지 않는다.

**필수 contextFiles**:

없음. 감사 에이전트는 실행 시점에 직접 디렉토리를 스캔하여 파일 인벤토리를 구축한다(Phase 0: Discovery). contextFiles를 주입하면 "실행 시점 스캔" 원칙이 우회되어 감사 신뢰성이 훼손된다.

감사 결과는 `.tmp/audit-*.md`에 기록되며, 메인 오케스트레이터는 `.tmp/audit-final.md`를 읽어 후속 조치를 결정한다.
