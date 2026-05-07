# Playbook: QA Reporter (오류 리포트 형식)

오류 발생 시 아래 형식으로 리포트를 작성한다.

## 필수 포함 항목
1. **환경 버전** — ashfox 버전, Blockbench 버전, Claude Code 버전
2. **재현 단계** — 오류를 재현하기 위한 단계별 절차
3. **요청/응답 JSON 이력** — verbatim (그대로) 복사
4. **Revision 체인** — 어떤 순서로 revision이 변경됐는지
5. **기대값 vs 실제값** — 무엇을 기대했고 무엇이 나왔는지
6. **증거 경로** — trace 파일, preview 이미지, 텍스처 메타데이터
7. **추정 원인** — "가설:" 레이블 명시 후 작성

## 리포트 예시
```
## QA Report

### 환경
- ashfox: 0.0.4
- Blockbench: 5.1.4

### 재현 단계
1. ...
2. ...

### 요청 JSON
{ ... }

### 응답 JSON
{ ... }

### Revision 체인
c27e0828 → 32bd0b7e → ...

### 기대값 vs 실제값
기대: changedPixels=64
실제: isError=true, message="invalid texture op"

### 추정 원인 (가설)
fill_rect 사용 시 width/height 미지정으로 인한 파라미터 누락
```
