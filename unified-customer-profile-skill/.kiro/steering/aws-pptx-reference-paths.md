---
inclusion: always
---

# aws-pptx-generator 스킬 경로 매핑

SKILL.md와 references 문서에서 등장하는 상대 경로를 항상 스킬 루트 기준으로 resolve한다.
글로벌 `skill-reference-resolution` 규칙에 **이 스킬의 구체적 매핑**을 덧붙이는 스티어링.

스킬이 워크스페이스 `.kiro/skills/` 하위에 배포되어 있으므로 모든 경로는 `.kiro/skills/aws-pptx-generator/`를 루트로 한다.

## 경로 매핑 표

| SKILL.md 또는 references에서 언급 | 실제 경로 (워크스페이스 루트 기준)                       |
| --------------------------------- | -------------------------------------------------------- |
| `references/xxx.md`               | `.kiro/skills/aws-pptx-generator/references/xxx.md`      |
| `scripts/xxx.py`                  | `.kiro/skills/aws-pptx-generator/scripts/xxx.py`         |
| `scripts/schemas/xxx.py`          | `.kiro/skills/aws-pptx-generator/scripts/schemas/xxx.py` |
| `scripts/slides/xxx.py`           | `.kiro/skills/aws-pptx-generator/scripts/slides/xxx.py`  |
| `tokens/xxx.yaml`                 | `.kiro/skills/aws-pptx-generator/tokens/xxx.yaml`        |
| `assets/aws-icons/...`            | `.kiro/skills/aws-pptx-generator/assets/aws-icons/...`   |
| `assets/backgrounds/...`          | `.kiro/skills/aws-pptx-generator/assets/backgrounds/...` |
| `subagents/xxx.md`                | `.kiro/skills/aws-pptx-generator/subagents/xxx.md`       |
| `hooks/xxx.kiro.hook`             | `.kiro/skills/aws-pptx-generator/hooks/xxx.kiro.hook`    |
| `steering/xxx.md`                 | `.kiro/skills/aws-pptx-generator/steering/xxx.md`        |

## 명령 호출 관례

모든 python3 호출은 **워크스페이스 루트에서** 아래 형태로 수행한다.

```bash
python3 .kiro/skills/aws-pptx-generator/scripts/check_env.py --install
python3 .kiro/skills/aws-pptx-generator/scripts/validate.py .pptx-generator/slides/
python3 .kiro/skills/aws-pptx-generator/scripts/validate_md.py .pptx-generator/slides/
python3 .kiro/skills/aws-pptx-generator/scripts/validate_svg.py .pptx-generator/slides/
python3 .kiro/skills/aws-pptx-generator/scripts/embed_icons.py <svg_file>
python3 .kiro/skills/aws-pptx-generator/scripts/render_to_png.py <pptx_file> -o <render_dir>
python3 .kiro/skills/aws-pptx-generator/scripts/build.py <slides_dir> <output_path>
```

`cd .kiro/skills/aws-pptx-generator/scripts/ && python3 xxx.py` 같은 cwd 변경 실행 금지 — 상대 경로 기준이 바뀌어 다른 아티팩트를 못 찾는다.

## SVG 내 `<image href>` 속성

SVG가 워크스페이스 루트에서 해석될 때 아이콘 경로도 워크스페이스 루트 기준이어야 한다. 그래서 href는:

```xml
<image href=".kiro/skills/aws-pptx-generator/assets/aws-icons/service/Arch_AWS-Lambda_48.png" .../>
```

이 경로로 SVG 작성 → `embed_icons.py` 실행 → base64 data URI로 변환된 `-embedded.svg`가 최종 산출물.

이 규칙을 놓치면 JSON validator(`validate.py`)의 아이콘 경로 계약 검증에서 실패한다 (임베딩 0개 + PPTX 렌더 시 아이콘 누락).

## 산출물 경로와의 구분

스킬 자체(`.kiro/skills/aws-pptx-generator/`)는 **읽기 전용**. 파이프라인 산출물은 반드시 `.pptx-generator/` 하위(일반) 또는 `.pptx-generator/stress-test/` 하위(스트레스 테스트)에 생성한다.

절대 스킬 내부 디렉토리에 산출물을 쓰지 않는다 — 스킬 감사 시 혼란을 만든다.
