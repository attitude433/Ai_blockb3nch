# Playbook: Texture Quality (텍스처 품질 검증)

`paint_faces` 호출 후 반드시 아래 항목을 확인한다.

## 체크리스트
- [ ] `targets=1` — 정확히 1개 큐브를 대상으로 했는지
- [ ] `opsApplied=1` — 정확히 1개 작업이 적용됐는지
- [ ] `changedPixels` — 기대한 픽셀 수와 일치하는지
- [ ] `resolvedSource` — 의도한 좌표 공간(`face` 기본값)인지
- [ ] `validate` — 새로운 UV 문제가 생기지 않았는지
- [ ] `read_texture` hash/byteLength — 정상 범위인지

## 이상 징후 (즉시 중단)
- byteLength가 급격히 감소
- hash가 갑자기 리셋
- preview가 빈 화면으로 나옴

이상 징후 발생 시 → 추가 쓰기 중단 + QA 리포트 작성

## paint_faces 사용 시 주의사항
- `coordSpace="face"` 명시 필요
- `x, y, width, height` 값 명시 필요 (`fill_rect` 사용 시)
- 한 번에 하나의 face, 하나의 op만 처리
