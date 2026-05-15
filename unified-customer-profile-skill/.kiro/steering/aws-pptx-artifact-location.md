---
inclusion: always
---

# aws-pptx-generator 아티팩트 위치 관례

파이프라인이 끊기지 않도록 모든 아티팩트를 정해진 위치에만 생성한다. 절대 경로 사용 금지 — 모두 워크스페이스 루트 기준 상대 경로.

## 설계 원칙

- **`.pptx-generator/`는 사용자 최종 산출물 전용**. 중간 로그·임시 분석·감사 리포트는 여기 두지 않는다
- **`.tmp/`는 임시·디버그 산출물 전용**. 서브에이전트 배치 결과, 시각 분석 보고서, 감사 리포트 등 파이프라인 재현·디버깅용 파일은 전부 여기로 간다
- **`logs/` 디렉토리는 사용하지 않는다**. 과거에 `.pptx-generator/logs/`를 두었던 설계는 `.tmp/`와의 이중 구조를 만들어 일관성을 깨뜨렸으므로 폐기

## 일반 모드 경로

```
.pptx-generator/
├── analysis.md                         ← Phase 1
├── questions.md                        ← Phase 1
├── scenario.md                         ← Phase 2 (output_path 메타 포함)
├── slides/
│   ├── slide-NN.md                     ← Phase 3 (모든 타입 — Cover/Agenda/Section/Content/ThankYou)
│   ├── slide-NN.json                   ← Phase 4/5
│   ├── slide-NN-[이름].svg             ← Phase 5 원본 SVG
│   └── slide-NN-[이름]-embedded.svg    ← Phase 5 임베딩 완료 SVG (최종 참조 대상)
├── [발표제목].pptx                     ← Phase 6 (파일명은 scenario.md의 output_path)
└── renders/
    └── slide-NN.png                    ← Phase 7 (선택)
```

## 스트레스 테스트 모드 경로

```
.pptx-generator/stress-test/
├── analysis.md
├── questions.md
├── scenario.md
├── slides/
│   ├── slide-NN.md
│   ├── slide-NN.json
│   └── slide-NN-*-embedded.svg
├── stress-test-[YYYYMMDD-HHMM].pptx    ← Phase 6 (타임스탬프 자동 생성)
├── renders/
│   └── slide-NN.png                    ← Step 10 시각 분석용 PNG
├── issues/
│   └── issue-NNN.md
├── coverage-matrix.md
├── fixes-applied.md
└── final-report.md
```

산출물이 평탄화되어 있다. 이전 버전의 `run-A-simple/`, `run-B-medium/`, `run-C-complex/` 하위 분할 구조는 폐기됐다. 스트레스 테스트 중간 로그(시각 분석 보고서 등)는 `.tmp/` 아래에 기록된다 — 아래 "임시 산출물" 섹션 참조.

## 임시 산출물 (`.tmp/`)

서브에이전트 결과·시각 분석·감사 리포트 등 파이프라인 재현·디버깅용 파일은 모두 `.tmp/`에 둔다.

```
.tmp/
├── phase3-slides-[범위]-result.md      ← Phase 3 서브에이전트 배치 결과 (일반)
├── phase5-slides-[범위]-result.md      ← Phase 5 서브에이전트 배치 결과 (일반)
├── visual-analysis-*.md                ← Phase 7 시각 분석 보고서
├── audit-*.md                          ← pptx-skill-auditor 감사 리포트
├── pptx-test/
│   └── *.py                            ← 스트레스 테스트용 임시 검수 스크립트
└── stress-test/
    ├── phase3-slides-[범위]-result.md  ← Phase 3 배치 결과 (스트레스 모드)
    ├── phase5-slides-[범위]-result.md  ← Phase 5 배치 결과 (스트레스 모드)
    ├── visual-analysis-range-*.md      ← Step 10 청크 보고서
    ├── visual-analysis-iter-*.md       ← Step 10 반복 분석 보고서
    └── visual-analysis-integrated-iter-*.md  ← Step 10 청크 통합 리포트
```

일반 모드와 스트레스 모드의 서브에이전트 결과 파일은 `.tmp/stress-test/`와 `.tmp/` 루트로 분리된다. 파일명은 같아도 디렉토리가 다르므로 충돌하지 않는다.

## PPTX 출력 위치

- **일반 모드**: `.pptx-generator/[파일명].pptx` (산출물 영역 안). 파일명은 `scenario.md`의 `output_path` 메타에서 읽는다. 이 값은 Phase 1 `questions.md` 질문 10의 답변에서 유래한다.
- **스트레스 테스트 모드**: `.pptx-generator/stress-test/stress-test-[YYYYMMDD-HHMM].pptx` (타임스탬프 자동 생성)

`output.pptx` 같은 고정 파일명은 사용하지 않는다. 사용자가 여러 번 빌드할 때 덮어쓰기·구분 불가 문제를 피하기 위함이다.

## 금지 행동

- `./slides/`, `output/`, `pptx-output/` 같은 임의 디렉토리에 산출물 생성 금지
- `.pptx-generator/slides/`를 일반 모드와 스트레스 테스트 모드가 공유하지 않음 — 스트레스 테스트는 반드시 `stress-test/slides/` 하위 사용
- 스킬 내부(`aws-pptx-generator/`) 파일을 파이프라인 산출물로 덮어쓰지 않음 — 스킬 자체는 읽기 전용
- **`.pptx-generator/logs/` 디렉토리 생성 금지** — 임시 산출물은 전부 `.tmp/`로

## 경로 결정 우선순위

1. 사용자가 명시적으로 경로를 지정하면 그 경로 사용
2. 스트레스 테스트 트리거면 `.pptx-generator/stress-test/` 사용
3. 그 외에는 `.pptx-generator/` 루트 사용
