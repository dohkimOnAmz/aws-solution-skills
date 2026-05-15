---
inclusion: fileMatch
fileMatchPattern: ".pptx-generator/**/slide-*.svg"
---

# aws-pptx-generator SVG 임베딩 강제

`svg_to_slide.py`의 `_add_image`는 **base64 data URI만 지원**한다. 원본 상대 경로 SVG는 경고 출력 후 skip되어 빌드된 PPTX에 아이콘이 포함되지 않는다.

## 임베딩 실행 주체

**파일 쓰기 훅은 없다.** 분할 쓰기 중 미완성 SVG에 embed_icons.py가 돌면 XML 파싱 실패, 무한 트리거 위험 등이 있어 훅 방식은 의도적으로 폐기됐다.

대신 `pptx-slide-builder` 서브에이전트가 `pptx-slide-builder.md` BLOCKING 규칙 7번에 따라 **SVG 작성 종료 후** `embed_icons.py`를 직접 실행하여 `-embedded.svg`를 생성한다.

## 규칙

1. **원본 SVG 저장 → 즉시 embed_icons.py 실행**: `slide-NN-[이름].svg` 작성이 끝난 직후 서브에이전트가 직접 `python3 .kiro/skills/aws-pptx-generator/scripts/embed_icons.py <svg>`를 실행한다
2. **JSON은 embedded 참조**: `add_diagram` fn의 `file` 파라미터에는 반드시 `slide-NN-[이름]-embedded.svg`를 지정한다. 원본 `.svg` 참조 금지
3. **SVG 먼저, JSON 나중에**: JSON 작성 전에 SVG + embed가 완료되어 있어야 한다 — JSON이 참조하는 경로의 파일이 실재해야 validate.py가 통과한다
4. **embed 실패 시 즉시 수정**: embed_icons.py가 에러를 내면 원본 SVG의 `<image>` 태그 형식과 아이콘 경로를 재확인하고 재시도

## 임베딩 실패 유형과 대응

| 실패 유형                  | 대응                                                                        |
| -------------------------- | --------------------------------------------------------------------------- |
| 아이콘 파일 경로 오타      | `icon-catalog.md`에서 정확한 파일명 확인 후 SVG 수정                        |
| 아이콘 파일 실존 안 함     | `.kiro/skills/aws-pptx-generator/assets/aws-icons/` 하위에서 유사 이름 검색 |
| base64 변환 시 메모리 초과 | 아이콘 개수를 줄이거나 ICON_S 크기로 축소                                   |
| `-embedded.svg` 생성 실패  | 원본 SVG의 XML 파싱 에러 — `<image>` 태그 형식 재확인                       |

## JSON 예시

올바름:

```json
{
  "fn": "add_diagram",
  "args": {
    "file": "slide-08-aws-arch-embedded.svg",
    "x": "C1",
    "y": "R2"
  }
}
```

잘못됨 (빌드되지만 아이콘 누락):

```json
{
  "fn": "add_diagram",
  "args": {
    "file": "slide-08-aws-arch.svg",
    "x": "C1",
    "y": "R2"
  }
}
```

## 금지 행동

- JSON에서 원본 `.svg` 참조
- `-embedded.svg` 생성 없이 JSON 작성 완료 선언
- `-embedded.svg` 생성 없이 `build.py` 실행
- `embed_icons.py` 실패 로그 무시
- 서브에이전트가 "embed는 나중에" 하고 JSON을 먼저 저장
