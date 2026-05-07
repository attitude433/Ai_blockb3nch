# Playbook: Mutation Contract (변경 작업 순서)

모든 ashfox 뮤테이션 호출에 이 순서를 따른다.

## 필수 순서
1. `get_project_state` 호출 → `project.revision` 캡처
2. `ifRevision` 포함하여 변경 1건 실행
3. 응답 검증
   - 성공: revision이 변경됨 (또는 `no_change`)
   - 실패: 전체 에러 페이로드 캡처
4. `get_project_state` 재호출 → 사후 조건 확인

## 필수 규칙
- `ifRevision` 없는 뮤테이션 호출 금지
- 원자적으로 실행: 호출 1번 = 의도 1개
- revision 재확인 없이 연속 쓰기 금지
- `invalid_state_revision_mismatch` 발생 시 → 새 revision 조회 후 1회만 재시도

## 고위험 작업 (추가 검증 필수)
| 도구 | 추가 검증 |
|------|----------|
| `paint_faces` | `changedPixels`, `resolvedSource`, 텍스처 hash/byteLength 확인 |
| `add_cube` / `update_cube` | `validate` 실행 후 UV 경고 비교 |
| `set_frame_pose` | preview 렌더링으로 뷰포트 반영 확인 |
