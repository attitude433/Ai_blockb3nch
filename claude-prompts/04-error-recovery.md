# Playbook: Error Recovery (오류 복구)

## `invalid_state_revision_mismatch`
1. `get_project_state` 호출
2. `ifRevision`을 최신 revision으로 교체
3. 1회 재시도

## `invalid_payload` (스키마 오류)
1. `details.path`와 `details.rule` 확인
2. 페이로드 형태만 수정 (범위는 그대로 유지)
3. 동일 의도로 재시도

## `invalid_state` (UV 관련)
원인: `uv_overlap`, `uv_scale_mismatch`, `uv_usage_mismatch`
1. `validate` 실행
2. 필요 시 UV 복구 경로 실행 (`autoUvAtlas`)
3. 단일 target/face/op 로 재페인트

## 텍스처 이상 징후
신호: byteLength 급감, hash 리셋, 빈 preview
1. 추가 쓰기 즉시 중단
2. trace와 raw 요청/응답 수집
3. QA 리포트 작성

## 오늘 실제로 겪은 오류 메모
- `invalid texture op` → `fill_rect` 사용 시 `x, y, width, height` 명시 필요
- `$.ifRevision must be string` → 함수 반환값이 null일 때 다음 호출 실패 (에러 핸들링 필수)
- PowerShell `AC` 충돌 → ashfox 함수명에 PowerShell 예약어 사용 금지
- PowerShell `$args` 충돌 → 함수 파라미터명으로 `$args` 사용 금지
