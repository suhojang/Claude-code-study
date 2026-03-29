---
name: review-with-codex
description: Open a cmux split pane to have Codex (GPT 5.4 xhigh fast) review any plan (plan mode, /plan, superpowers, ralph, or file-based), then feed the review back to rethink and improve the plan.
disable-model-invocation: true
allowed-tools: Bash(cmux *), Bash(cat *), Bash(sleep *), Bash(wc *), Bash(ls *), Bash(test *), Read, Write, Edit, Glob, Grep
argument-hint: "[file-path]"
---

# Review Any Plan with Codex via cmux Split

어떤 방식으로 작성된 플랜이든 cmux 화면분할로 Codex(GPT 5.4 xhigh fast)에게
리뷰받고, 피드백을 바탕으로 누락/보완 사항을 반영하여 플랜을 재작성한다.

## 지원하는 플랜 소스

| 소스 | 플랜 위치 | 캡처 방법 |
|:---|:---|:---|
| Plan mode (Shift+Tab) | 화면 출력 | `cmux read-screen` |
| `/plan` 스킬 | 화면 출력 | `cmux read-screen` |
| Superpowers | 화면 출력 | `cmux read-screen` |
| Ralph plan 등 커뮤니티 스킬 | 파일 (`docs/plans/*.md` 등) | 파일 직접 읽기 |
| 사용자 지정 파일 | 인수로 전달 | 파일 직접 읽기 |

## Workflow

### 1. cmux 환경 확인

```bash
if [ -z "$CMUX_WORKSPACE_ID" ]; then
  echo "ERROR: cmux 환경이 아닙니다. cmux 터미널에서 실행해주세요."
  exit 1
fi
```

### 2. 플랜 소스 자동 감지 및 캡처

**우선순위에 따라 플랜 소스를 결정한다:**

#### Case A: 인수로 파일 경로가 주어진 경우
`$ARGUMENTS`가 존재하고 해당 파일이 실제로 존재하면 그 파일을 플랜으로 사용한다.

```bash
# 예: /review-with-codex docs/plans/order-cancel.md
cp "$ARGUMENTS" /tmp/current-plan.md
```

#### Case B: 인수 없음 → 최근 플랜 파일 탐색
`docs/plans/` 등 일반적인 플랜 저장 경로에서 **최근 5분 이내 수정된** `.md` 파일을 찾는다.
Ralph plan 등 파일 기반 플랜 도구가 여기에 해당한다.

```bash
find docs/plans/ -name "*.md" -mmin -5 -type f 2>/dev/null | head -1
```

파일이 발견되면 해당 파일을 플랜으로 사용한다.

#### Case C: 파일 없음 → 현재 화면에서 캡처 (기본)
Plan mode, `/plan`, Superpowers 등 화면에 출력된 플랜을 캡처한다.
**대부분의 경우 여기에 해당한다.**

```bash
cmux read-screen --lines 500 > /tmp/current-plan-raw.txt
```

캡처 후 플랜 본문만 추출한다:
- 터미널 프롬프트 (`$`, `❯`, `claude>` 등) 제거
- Claude Code UI 프레임 (─, │, ╭, ╰ 등) 제거
- 도구 호출 로그, 승인 프롬프트 등 제거
- **마크다운 헤더(`#`), 목록(`-`, `1.`), 본문 텍스트**만 남긴다
- 추출한 내용을 `/tmp/current-plan.md`에 Write 도구로 저장

### 3. 캡처 결과 검증

```bash
wc -l /tmp/current-plan.md
```

- 10줄 미만이면 캡처 실패로 판단하고 사용자에게 알린다
- 충분한 내용이 있으면 다음 단계로 진행

### 4. 오른쪽에 Codex pane 자동 생성

```bash
cmux new-split right
sleep 1
cmux list-workspaces
```

새로 생성된 surface reference를 확인한다.

### 5. Codex에 리뷰 요청 전송

```bash
cmux send --surface <new-surface> "codex --model gpt-5.4-xhigh-fast --quiet 'You are a senior architect reviewer. Read and review the implementation plan in /tmp/current-plan.md thoroughly.\n\nFocus on:\n1. Missing edge cases or error scenarios\n2. Architecture or design concerns\n3. Performance bottlenecks\n4. Security vulnerabilities\n5. Testing gaps\n6. Dependencies or ordering issues between tasks\n7. Unclear or ambiguous requirements\n\nBe specific and actionable. For each issue explain WHY it matters and suggest a concrete fix.\n\nWrite your complete review to /tmp/codex-plan-review.md'\n"
```

### 6. 리뷰 완료 대기

```bash
cmux set-progress 0.3 --label "Codex reviewing plan..."
```

완료 판별 (5초 간격 반복):
1. `/tmp/codex-plan-review.md` 파일 존재 및 내용 확인
2. `cmux read-screen --surface <new-surface> --lines 10`으로 Codex 종료 확인

```bash
cmux set-progress 0.7 --label "Review received, analyzing..."
```

### 7. 리뷰 결과를 현재 pane으로 가져오기

**데이터 전달 메커니즘**: Pane A(현재 Claude Code)와 Pane B(Codex)는
같은 파일시스템(`/tmp/`)을 공유한다. Codex가 파일에 쓰면, 현재 pane에서 읽을 수 있다.

```bash
cat /tmp/codex-plan-review.md
```

이 Bash 도구 실행 결과가 **현재 Claude Code 세션의 컨텍스트에 직접 주입**된다.
즉, Claude가 리뷰 전문을 읽고 이해한 상태에서 다음 단계를 수행한다.

### 8. 플랜 재작성 (핵심)

**현재 pane(원래 플랜을 작성하던 Claude Code 세션)에서** 리뷰를 바탕으로 플랜을 다시 생각한다:

- 리뷰에서 지적한 **누락된 내용** 추가
- **아키텍처 우려사항** 반영하여 설계 수정
- **엣지 케이스**와 **에러 시나리오** 보강
- **테스트 전략** 보완
- **태스크 순서/의존성** 재정렬

#### 출력 방식 (플랜 소스에 따라 다름)

**Case A, B (파일 기반 플랜)**:
원본 파일을 직접 수정한다. 상단에 리뷰 반영 요약을 추가한다.

**Case C (화면 기반 플랜)**:
재작성된 플랜을 화면에 출력한다. 파일로 저장하지 않는다.

공통으로 상단에 다음을 포함한다:

```
## Codex Review 반영 사항 (GPT 5.4 xhigh fast)
- [누락되어 추가한 항목들]
- [수정/보강한 항목들]
- [Codex가 지적했지만 반영하지 않은 항목과 그 이유]

---
(재작성된 플랜 본문)
```

### 9. 완료 알림

```bash
cmux set-progress 1.0 --label "Plan rewritten with Codex review"
cmux notify --title "Plan Review Complete" --body "Plan has been rewritten based on Codex feedback."
```

## Important Notes

- 플랜 소스를 자동 감지한다: 인수 → 최근 파일 → 화면 캡처 순
- Codex pane은 리뷰 후에도 닫지 않는다 (사용자가 직접 확인 가능)
- 캡처 원본: `/tmp/current-plan-raw.txt`
- 정리된 플랜: `/tmp/current-plan.md`
- 리뷰 원본: `/tmp/codex-plan-review.md`
- 이 스킬은 현재 pane(Claude Code)에서 실행되며, Codex는 리뷰어 역할만 수행한다
