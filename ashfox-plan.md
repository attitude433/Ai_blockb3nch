# ashfox 사용 계획

## 목표
Blockbench에서 AI로 마인크래프트 모델/텍스처/애니메이션 제작

## 사용 도구
- **Blockbench** (데스크톱 본체)
- **ashfox** (Blockbench 플러그인, 로컬 MCP 서버 제공)
  - https://github.com/sigee-min/ashfox
- **Claude Code** (MCP 클라이언트, 자연어로 ashfox 도구 호출)

## 구조
```
[사용자 자연어 명령]
        ↓
[Claude Code]
        ↓ (MCP 프로토콜)
[ashfox 로컬 서버 127.0.0.1:8787]
        ↓
[Blockbench 조작]
        ↓
[.bbmodel / GeckoLib 포맷 출력]
```

## 설치 순서
1. Blockbench 설치
2. ashfox 플러그인 설치 (Blockbench 안에서)
3. Claude Code 설치
4. Claude Code에 ashfox MCP 서버 등록

## 비용
- Claude Code 구독료 한도 내 사용 (API 별도 결제 X)

## 확인 필요한 것
- Claude Code가 ashfox와 실제로 잘 호환되는지 (README는 GPT-5.2 기준 테스트)
- 모델링 품질 (Claude의 3D 공간 추론 능력)
