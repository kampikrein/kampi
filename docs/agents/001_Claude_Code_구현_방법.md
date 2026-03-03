# Claude Code에서 Kampi 페르소나 구현 방법

> **핵심 결론**: CLAUDE.md가 페르소나 로드의 핵심 진입점이다. `@파일경로` 참조로 외부 YAML을 포함하고, 메모리 시스템으로 세션 간 연속성을 보장한다.

---

## 1. CLAUDE.md 패턴

### 1.1 역할과 동작 원리

CLAUDE.md는 Claude Code가 모든 세션에서 **자동으로 로드**하는 프로젝트 지시 파일이다.

```
세션 시작
   ↓
CLAUDE.md 자동 로드
   ↓
@참조 파일 포함 (persona/kampi.yaml)
   ↓
MEMORY.md 로드 (자동 메모리 시스템)
   ↓
페르소나 활성화
```

### 1.2 CLAUDE.md에서 외부 파일 참조

```markdown
## 페르소나

@persona/kampi.yaml
```

이 단순한 참조가 kampi.yaml의 전체 내용을 CLAUDE.md에 포함시킨다.

### 1.3 CLAUDE.md 작성 모범 사례

```markdown
# 프로젝트명

## 페르소나
@persona/kampi.yaml

## 프로젝트 컨텍스트
[프로젝트 배경 설명]

## 작업 규칙
[구체적인 작업 지침]
```

**주의사항**:
- CLAUDE.md는 200줄 이하로 유지 (더 길면 잘림)
- 핵심만 직접 기술, 나머지는 @참조로 분리
- 자주 변하는 정보는 별도 파일로 관리

---

## 2. 메모리 시스템

### 2.1 Auto Memory 디렉토리

Claude Code는 프로젝트별 자동 메모리 디렉토리를 제공:

```
~/.claude/projects/[프로젝트경로해시]/memory/
   ├── MEMORY.md    ← 항상 자동 로드 (200줄 이하 권장)
   ├── patterns.md  ← MEMORY.md에서 링크하는 상세 파일
   └── ...
```

### 2.2 MEMORY.md 구조 권장

```markdown
# 프로젝트 메모리

## 핵심 컨텍스트
[가장 중요한 맥락 - 항상 필요한 정보]

## 페르소나 상태
[Kampi 페르소나 관련 기록]

## 사용자 선호
[확인된 선호 패턴]

## 미결정 사항
[아직 결정 안 된 것들]
```

### 2.3 세션 간 정보 보존 전략

| 정보 종류 | 저장 위치 | 이유 |
|-----------|-----------|------|
| 페르소나 정의 | persona/kampi.yaml | 구조화된 형태 |
| 사용자 선호 | MEMORY.md | 자동 로드 |
| 프로젝트 상태 | MEMORY.md | 자동 로드 |
| 상세 인사이트 | docs/별도파일.md | MEMORY.md에서 링크 |

---

## 3. Hooks 활용

### 3.1 Claude Code Hooks 개요

Hooks는 특정 이벤트 발생 시 자동으로 실행되는 스크립트:

```json
{
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...],
    "Stop": [...]
  }
}
```

### 3.2 페르소나 관련 Hooks 활용 예시

**세션 종료 시 자동 메모리 업데이트** (Stop hook):
```json
{
  "hooks": {
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "echo '세션 종료: $(date)' >> ~/.claude/session_log.txt"
      }]
    }]
  }
}
```

**주의**: Hooks는 `~/.claude/settings.json`에 설정 (프로젝트 CLAUDE.md가 아님)

---

## 4. Subagent 및 Skill 활용

### 4.1 페르소나 시스템에서 Subagent 역할

```
Kampi (메인 세션)
   ├── Explore Agent: 코드베이스 탐색
   ├── Plan Agent: 실행 계획 수립
   └── General-purpose Agent: 리서치 작업
```

서브에이전트는 독립적인 컨텍스트로 실행되므로 **페르소나가 전파되지 않음**.
메인 세션에서만 Kampi 페르소나가 활성화된다.

### 4.2 Skills (Slash Commands) 활용

프로젝트 루트 또는 `~/.claude/` 에 커스텀 스킬 정의 가능:

```markdown
<!-- .claude/commands/vibe-check.md -->
# 바이브 체크
현재 작업 흐름과 에너지 레벨을 확인하고
다음 집중 포인트를 제안합니다.
```

사용: `/vibe-check`

### 4.3 페르소나 전용 Skills 아이디어

| 스킬명 | 역할 |
|--------|------|
| `/kampi-plan` | 프로젝트 상태 검토 + 다음 액션 제안 |
| `/kampi-review` | 완성된 작업물 솔직한 피드백 |
| `/kampi-sync` | 메모리 + 페르소나 상태 동기화 |

---

## 5. 구현 아키텍처 권장안

### 5.1 파일 구조

```
/youtube/
├── CLAUDE.md              ← 페르소나 참조 + 프로젝트 지침
├── persona/
│   └── kampi.yaml         ← 페르소나 정의 (메인)
├── docs/
│   └── 001_설계보고서.md
└── ...
```

### 5.2 로드 순서

```
1. CLAUDE.md 읽기
2. @persona/kampi.yaml 포함
3. ~/.claude/projects/.../memory/MEMORY.md 자동 로드
4. 페르소나 앵커링 완료
```

### 5.3 핵심 구현 원칙

1. **단일 진실 원천**: 페르소나 정의는 kampi.yaml 하나에만
2. **CLAUDE.md는 게이트**: 진입점만 담고, 세부사항은 참조
3. **MEMORY.md는 상태**: 누적 컨텍스트와 진화 기록
4. **테스트 가능성**: 새 세션에서 페르소나 자동 활성화 확인

---

*작성: 에이전트 2 (Claude Code 구현 방법) | 2026-03-03*
