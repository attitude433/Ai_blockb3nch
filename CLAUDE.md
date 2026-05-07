# CLAUDE.md - ashfox + Blockbench 작업 가이드

## 프로젝트 개요
Claude Code + ashfox MCP를 통해 Blockbench에서 마인크래프트 모델/텍스처/애니메이션을 제작한다.

## 구조
```
[사용자 자연어 명령]
        ↓
[Claude Code]
        ↓ (MCP - http://127.0.0.1:8787/mcp)
[ashfox 로컬 서버]
        ↓
[Blockbench 실시간 조작]
```

## 환경
- Blockbench: 5.1.4
- ashfox: 0.0.4
- MCP 엔드포인트: `http://127.0.0.1:8787/mcp`
- 기본 포맷: GeckoLib (애니메이션 지원)

## 작업 시작 전 필수 확인
1. Blockbench가 실행 중인지 확인
2. ashfox 서버 응답 확인: `127.0.0.1:8787` 포트 열려 있는지
3. `get_project_state` → 현재 revision 캡처

---

## 제작자 권장 작업 파이프라인 (4패스)

모든 모델 제작은 아래 순서를 따른다. (출처: agents/codex/skills/ashfox-operator/SKILL.md)

### Safe Pass — 변경 안전 규칙
1. `get_project_state` → `project.revision` 캡처
2. 뮤테이션 1회 실행 (`ifRevision` 포함)
3. 응답 검증 (revision 변경 확인)
4. `get_project_state` 재호출 → 사후 조건 확인

### Constraint Pass — 구조 제약 검증
- `humanoid-face` 프로파일: 눈 좌우 대칭, 코·입 위치 순서
- `biped-foot` 프로파일: 발 지면 접촉, 발가락 전방 방향, 좌우 간격
- 제약 실패 시 → Repair Pass 실행

### Repair Pass — 제약 오류 수정
- 실패한 규칙 하나당 뮤테이션 1회
- 최대 2회 반복 후에도 실패 시 → 작업 중단 + 실패 진단 반환

### Craft Pass — 시각 품질 향상
1. 스타일 프로파일 선택 (아래 참조)
2. `render_preview` 실행
3. 품질 루브릭 점수 산정
4. 집중 개선 뮤테이션 1회
5. 재렌더 + 재점수
- 최대 2회 반복

---

## 스타일 프로파일 (Craft Pass에서 선택)

### hard-surface-voxel (갑옷·기계·건축물)
- 좁은 팔레트 + 단일 액센트 계열
- 패널 경계 강조, 엣지 하이라이트 중심
- 그라데이션 최소화, 기능적 컴포넌트 간 대비
- 실패 신호: 팔레트 혼탁, 패널 경계 불명확

### minecraft-natural-shade (바닐라 스타일)
- 중간 채도 유지, 순수 흑/백 사용 금지
- 밝은 면 따뜻하게, 어두운 면 차갑게 (소폭 색조 이동)
- 주요 면당 2~4 단계 톤
- 실패 신호: 대형 면이 단색 평면, 일관성 없는 광원 방향

### creature-organic (생물·유기체)
- 탁한 베이스 톤 + 핵심 부위만 고채도 액센트
- 근육·관절 볼륨 강조, 부드러운 그라데이션
- 해부학적 볼륨 우선, 작은 디테일 지양
- 실패 신호: 주요 근육군 간 볼륨 구분 없음

---

## 품질 루브릭 (Craft Pass 점수 기준)

0=사용 불가 / 1=주요 결함 / 2=눈에 띄는 결함 / 3=허용 기준 / 4=우수 / 5=프로덕션 수준

| 항목 | 기준 |
|------|------|
| 형태·비율 | 기본 앵글에서 실루엣이 의도한 대로 읽힘 |
| UV 안정성 | uv_overlap / uv_scale_mismatch 없음 |
| 텍스처 가독성 | 게임 거리에서 주요 형태 식별 가능 |
| 애니메이션 가독성 | (애니메이션 수정 시만) 키 포즈 명확 |
| 스타일 일관성 | 선택한 프로파일 팔레트·음영 일치 |

통과 기준: 전 항목 최소 3점, 평균 4점 이상. 3점 미만 항목 → 집중 개선 패스 1회 실행.

---

## 핵심 규칙 (반드시 준수)

### 1. 모든 변경은 revision 기반으로
- 변경 전 항상 `get_project_state` 호출
- 모든 뮤테이션 호출에 `ifRevision` 필수 포함
- 변경 후 새 revision 저장
- revision 없이 다음 호출 금지
- `invalid_state_revision_mismatch` → revision 갱신 후 1회만 재시도
- `invalid_payload` → payload 교정 없이 재시도 금지

### 2. 한 번에 하나만
- 호출 1번 = 의도 1개
- 연속 쓰기 시 매번 revision 갱신

### 3. paint_faces 주의사항
- 파라미터명: `target` (단수 객체), `op` (단수 객체) — `targets`/`ops` 배열 아님
- `target.face` 생략 시 해당 큐브의 **전체 6면** 한 번에 적용 (효율적)
- `coordSpace="face"` 반드시 명시
- `fill_rect` 사용 시 `x, y, width, height` 반드시 명시
- `coordSpace="texture"` 사용 시 최상위에 `width`, `height`도 필수

### 3-1. assign_texture 주의사항
- `cube` (문자열) 아님 — `cubeNames` (문자열 배열) 사용
- 특정 면만 할당할 때는 `faces` 배열 사용 가능
- 한 번에 여러 큐브에 할당: `cubeNames=@("a","b","c")`

### 4. 큐브 겹침 시 — inflate 우선 사용
- 몸통 큐브 위에 갑옷 큐브를 얹을 때는 **`inflate` 파라미터**를 사용한다
  ```
  add_cube "armor_torso" bone="chest" from=[-4,14,-3] to=[4,26,3] inflate=1.0
  ```
  → 모든 방향으로 1.0 단위 팽창. 수동 from/to 계산 불필요.
- inflate 없이 수동으로 겹칠 때는 **±0.001 오프셋** 필수 (Z-파이팅 방지)
  ```
  안쪽: from=[ 0,  0,  0]  to=[8, 8, 8]
  바깥: from=[-0.001,-0.001,-0.001]  to=[8.001,8.001,8.001]
  ```
- 권장 수동 범위: 0.001 ~ 0.01

### 5. 프로젝트 생성 순서
1. `ensure_project` — 반드시 `ifRevision` 포함, 새 프로젝트 강제 생성 시:
   ```
   match="format_and_name"  onMismatch="create"  onMissing="create"
   ```
2. `get_project_state` → revision·textures[0].id 캡처
3. 뼈대(bone) 추가
4. 큐브(cube) 추가 ← 텍스처 존재 후 추가해야 UV 자동 할당
5. `assign_texture` (cubeNames 배열로 한 번에)
6. `paint_faces` (target.face 생략 시 전체 면 적용)
7. `render_preview` — mode 필수: `"fixed"` 또는 `"turntable"`

---

## 전체 MCP 도구 목록 및 스펙

### 기본 도구
| 도구 | 주요 파라미터 | 비고 |
|------|-------------|------|
| `list_capabilities` | (없음) | 플러그인 지원 기능 목록 |
| `get_project_state` | `detail: 'summary'\|'full'`, `includeUsage` | `full`이면 geometry 포함 전체 상태 반환 |
| `read_texture` | `name`, `saveToTmp`, `tmpName` | hash/byteLength로 무결성 확인 |
| `export_trace_log` | `mode`, `destPath`, `fileName` | 디버그 트레이스 추출 |
| `reload_plugins` | `confirm: true`, `delayMs` | 플러그인 강제 재로드 |
| `validate` | (없음) | UV 겹침·불일치 검증 |

### 프로젝트 도구
| 도구 | 주요 파라미터 | 비고 |
|------|-------------|------|
| `ensure_project` | `format`, `target.name`, `match`, `onMismatch`, `onMissing`, `ifRevision` | format: `'geckolib'`(소문자 필수) |

### 모델 도구
| 도구 | 주요 파라미터 | 비고 |
|------|-------------|------|
| `add_bone` | `name`, `pivot[3]`, `parent`, `rotation[3]`, `scale[3]`, `visibility` | |
| `update_bone` | `name`, `newName`, `pivot`, `rotation`, `scale`, `parentRoot` | |
| `delete_bone` | `name` 또는 `names[]` | |
| `add_cube` | `name`, `from[3]`, `to[3]`, `bone`, `inflate`, `rotation[3]`, `origin[3]`, `mirror`, `uvOffset[2]` | **`inflate`**: 전방향 균등 팽창 (갑옷 오버레이에 활용) |
| `update_cube` | `name`, `newName`, `from`, `to`, `bone`, `inflate`, `rotation`, `origin` | |
| `delete_cube` | `name` 또는 `names[]` | |
| `add_mesh` | `name`, `bone`, `vertices[]`, `faces[]`, `origin`, `rotation`, `uvPolicy` | 비박스 폴리곤 메시. 곡면 형태에 활용 |
| `update_mesh` | `name`, `newName`, `vertices`, `faces`, `uvPolicy` | |
| `delete_mesh` | `name` 또는 `names[]` | |
| `export` | `format`, `destPath`, `options` | format: `gecko_geo_anim`, `gltf`, `java_block_item_json`, `animated_java` 등 |

### 텍스처 도구
| 도구 | 주요 파라미터 | 비고 |
|------|-------------|------|
| `assign_texture` | `textureName`, `cubeNames[]`, `faces[]` | faces 생략 시 전체 면 |
| `delete_texture` | `name` | |
| `paint_faces` | `target`, `op`, `coordSpace`, `ifRevision` | 아래 상세 참조 |
| `paint_mesh_face` | `target`, `op`, `coordSpace`, `scope` | 메시 면 페인팅 |

### 애니메이션 도구
| 도구 | 주요 파라미터 | 비고 |
|------|-------------|------|
| `create_animation_clip` | `name`, `length`, `loop`, `fps` | |
| `update_animation_clip` | `name`, `newName`, `length`, `loop`, `fps` | |
| `delete_animation_clip` | `name` 또는 `names[]` | |
| `set_frame_pose` | `clip`, `frame`, `bones[]` | bones 각 항목: `name`, `rot[3]`, `pos[3]`, `scale[3]`, `interp` |
| `set_trigger_keyframes` | `clip`, `channel`, `keys[]` | channel: `sound`, `particle`, `timeline` |

### 프리뷰 도구
| 도구 | 주요 파라미터 | 비고 |
|------|-------------|------|
| `render_preview` | `mode`, `angle[2~3]`, `output`, `saveToTmp`, `clip`, `timeSeconds`, `fps` | mode: `'fixed'`\|`'turntable'` 필수 |

---

## paint_faces 상세 스펙

### coordSpace
- `"face"` (기본): 각 면 로컬 좌표 (0,0)~(64,64) 기준
- `"texture"`: UV 텍스처 전체 좌표. 이 경우 최상위에 `width`, `height` 필수

### op 종류
| op | 설명 | 필수 파라미터 |
|----|------|-------------|
| `fill_rect` | 사각형 채우기 | `x, y, width, height, color` |
| `draw_rect` | 사각형 테두리선 | `x, y, width, height, color, lineWidth` |
| `draw_line` | 직선 | `x1, y1, x2, y2, color, lineWidth` |
| `set_pixel` | 단일 픽셀 | `x, y, color` |

### shade 파라미터 (fill_rect 전용)
단순 `shade=true/false` 외에 객체로 세밀하게 제어 가능:
```json
shade: {
  enabled: true,
  intensity: 0.4,        // 방향성 음영 강도 (0~1)
  edge: 0.3,             // 엣지 어두움 강도 (0~1)
  noise: 0.05,           // 픽셀별 랜덤 노이즈 (0~1)
  seed: 42,              // 결정론적 시드 (재현 가능)
  lightDir: "tl_br"      // tl_br | tr_bl | top_bottom | left_right
}
```
- 갑옷 질감: `intensity=0.4, edge=0.3, noise=0.05, lightDir="tl_br"`
- 바닐라 블록: `intensity=0.3, edge=0.1, noise=0.08`
- 단색 평면: `shade=false`

---

## add_mesh 스펙 (비박스 형태)

```json
add_mesh {
  name: "visor",
  bone: "helm",
  vertices: [
    { id: "v0", pos: [-3, 30, -5] },
    { id: "v1", pos: [ 3, 30, -5] },
    { id: "v2", pos: [ 3, 26, -5] },
    { id: "v3", pos: [-3, 26, -5] }
  ],
  faces: [
    { id: "f0", vertices: ["v0","v1","v2","v3"], texture: "Knight" }
  ],
  uvPolicy: { symmetryAxis: "x", texelDensity: 4, padding: 1 }
}
```

uvPolicy:
- `symmetryAxis`: `none | x | y | z`
- `texelDensity`: 0.25 ~ 64
- `padding`: 0 ~ 16

---

## set_frame_pose 상세 스펙

> ⚠️ **ashfox 0.0.4 버그**: `set_frame_pose`는 일부 본에만 적용됨. API는 성공 반환하지만 Blockbench 타임라인에 누락 발생. **사용 금지 — 아래 JSON 직접 작성 방식 사용.**

```json
set_frame_pose {
  clip: "walk",
  frame: 0,
  bones: [
    { name: "upper_arm_l", rot: [45, 0, 0], interp: "catmullrom" },
    { name: "upper_arm_r", rot: [-45, 0, 0], interp: "catmullrom" }
  ]
}
```

`interp`: `linear` | `step` | `catmullrom`

### frame 파라미터 — 초(second)가 아닌 프레임 번호
- `frame`은 시간(초)이 아니라 **프레임 인덱스**
- 실제 시간 = frame / snapping (`fps=20` → snapping=20)
- 0.25초 = frame **5**, 0.5초 = frame **10**, 0.75초 = frame **15**

---

## 애니메이션 제작 워크플로우 (확정)

`set_frame_pose`는 버그로 사용 불가. **JSON 파일 직접 작성**이 유일한 신뢰할 수 있는 방법.

### 파일 위치 및 네이밍
```
animations/{모델명}_{동작}.animation.json
예) animations/giant_walk.animation.json
```

### JSON 형식 (검증 완료)
```json
{
    "format_version": "1.8.0",
    "animations": {
        "animation.giant.walk": {
            "loop": true,
            "animation_length": 1,
            "bones": {
                "thigh_l": {
                    "rotation": {
                        "0.0":  { "post": { "vector": [35, 0, -3] }, "lerp_mode": "catmullrom" },
                        "0.25": { "post": { "vector": [ 5, 0, -3] }, "lerp_mode": "catmullrom" },
                        "0.5":  { "post": { "vector": [-30, 0, -3] }, "lerp_mode": "catmullrom" },
                        "0.75": { "post": { "vector": [ 5, 0, -3] }, "lerp_mode": "catmullrom" },
                        "1.0":  { "post": { "vector": [35, 0, -3] }, "lerp_mode": "catmullrom" }
                    }
                }
            }
        }
    }
}
```

### 핵심 규칙
1. **회전값 부호 반전**: Blockbench 기준값의 부호를 반전해서 입력 (검증: forearm_r [25,0,0] → JSON [-25,0,0])
2. **t=1.0 키프레임 필수**: t=0과 동일한 값으로 추가 — 없으면 루프 경계에서 catmullrom 탄젠트 계산 실패로 튐
3. **애니메이션 이름**: `animation.{모델명}.{동작}` 형식
4. **Load Animation File 불가**: Blockbench GeckoLib 플러그인의 Load Animation File은 키프레임 임포트가 아닌 저장 경로 연결만 함. JSON 파일은 게임 리소스에 직접 사용

---

## 텍스처 품질 게이트 (paint_faces 후 확인)

- **Gate A (쓰기 정상)**: `opsApplied=1`, `targets=1`, `changedPixels>0`
- **Gate B (좌표 정상)**: `resolvedSource.coordSpace="face"`, width/height 일치
- **Gate C (무결성)**: `validate` 실행 → uv_overlap 없음, `read_texture` hash/byteLength 정상
- **Gate D (시각)**: `render_preview`로 대상 면 육안 확인

이상 징후 (즉시 쓰기 중단):
- byteLength 급감
- hash 갑작스러운 리셋
- 프리뷰 빈 화면

---

## 오류 복구

| 오류 | 처리 방법 |
|------|---------|
| `invalid_state_revision_mismatch` | `get_project_state` → 새 revision → 1회 재시도 |
| `invalid_payload` (스키마 오류) | `details.path`, `details.rule` 확인 → payload만 수정 → 재시도 |
| `invalid_state` (UV 관련) | `validate` → 필요시 `autoUvAtlas` → 단일 target/face/op로 재페인트 |
| 텍스처 이상 징후 | 쓰기 즉시 중단 → trace 수집 → QA 리포트 |

---

## PowerShell 사용 시 주의
- 함수명에 `AC`, `SC`, `GC` 등 PowerShell 예약어 사용 금지
- 함수 파라미터명에 `$args` 사용 금지 (PowerShell 내장 변수와 충돌)
- 배열 연산은 hashtable 내부 계산 대신 변수에 미리 계산
- MCP 호출 시 JSON-RPC 2.0 형식 필수: `jsonrpc`, `id`, `method`, `params` 포함
  ```powershell
  $payload = @{ jsonrpc='2.0'; id=1; method='tools/call'; params=@{ name=$tool; arguments=$a } }
  ```

---

## 자주 범하는 실수 (과거 세션 기록)

| 실수 | 올바른 방법 |
|------|-----------|
| `targets`/`ops` 배열 사용 | `target`/`op` 단수 객체 |
| `cube` 파라미터 사용 | `cubeNames` 배열 |
| inflate 몰라서 수동 오프셋 계산 | `inflate=1.0` 사용 |
| shade=false만 알고 씀 | shade 객체로 intensity/edge/noise 세밀 제어 |
| draw_rect/draw_line 몰랐음 | 패널선·엣지선 그리기에 활용 |
| format="GeckoLib" 대문자 | format="geckolib" 소문자 필수 |
| ensure_project에 width/height 전달 | 해당 파라미터 없음. 텍스처 크기 설정 불가 |
| `$args` 파라미터명 사용 | `$a`, `$body` 등으로 변경 |
| AC 함수명 사용 | PowerShell 예약어. 전체 이름 사용 |
| 뼈대만 하라는 지시에 큐브까지 추가 | 지시 범위를 초과하지 않음 |
| `ensure_project`에 `target=@{name=...}` 전달 | `name` 필드는 최상위 — `name='MilkCarton'` (target 아님) |
| `add_mesh` GeckoLib 포맷에서 사용 시도 | GeckoLib는 메시 미지원. 큐브만 사용 가능 |
| MCP 호출 시 jsonrpc/id 필드 누락 | `@{ jsonrpc='2.0'; id=1; method='tools/call'; ... }` 필수 |
| 텍스처 미할당 상태에서 렌더 — 모델 안 보임 | 기본 텍스처가 흰색이라 흰 배경에 묻힘. 할당 후 페인팅 필수 |
| 큐브 회전(rotation) 방향 불명확 | GeckoLib 큐브 회전 부호 방향 검증 필요. 경사면은 계단식 슬래브로 근사하는 것이 안전 |
| `add_cube`에 `boxUv=true` 전달 | ashfox 0.0.4 미지원 — 무시되고 `boxUv=false`로 생성됨 |
| 모든 큐브를 `uvOffset=(0,0)`으로 생성 | UV 공간이 겹쳐 Z-파이팅과 동일한 투명 현상 발생. 큐브마다 `uvOffset`을 다르게 지정해 UV 영역이 겹치지 않게 할 것 |
| `paint_faces` 루프에서 revision 갱신 확인 안 함 | 호출마다 revision이 달라져야 정상. 연속 호출에서 같은 revision이 반복되면 이전 호출이 실패한 것 — SyncRev 후 재시도 |
| `update_cube` 후 재페인팅 누락 | update_cube는 내부 UV 아틀라스를 재배치하므로 새 UV 영역이 빈 상태로 생성됨. 수정 직후 해당 큐브 6면 재페인팅 필수 |
| `set_frame_pose` 사용 | ashfox 0.0.4 버그 — API는 성공 반환하지만 일부 본만 Blockbench에 반영됨. JSON 직접 작성 방식 사용 |
| `frame=0.25` 등 소수 사용 | frame은 초가 아닌 프레임 번호. fps=20이면 0.25s = frame 5 |
| 루프 애니메이션에 t=1.0 누락 | t=1.0 키프레임을 t=0과 동일하게 추가해야 루프 경계 매끄럽게 연결 |
| 검증 전 CLAUDE.md 수정 | 작동 확인 + 사용자 동의 후에만 기록 |
