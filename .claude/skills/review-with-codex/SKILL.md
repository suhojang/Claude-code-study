---
name: review-with-codex
description: Open a cmux split pane to have Codex (GPT 5.4 xhigh fast) review the current plan from screen output, then feed the review back so Claude can rethink and improve the plan.
disable-model-invocation: true
allowed-tools: Bash(cmux *), Bash(cat *), Bash(sleep *), Bash(wc *), Read, Write, Edit, Glob, Grep
---

# Review Current Plan with Codex via cmux Split

Superpowers(Claude Code)가 화면에 출력한 플랜을 cmux 화면분할로 Codex(GPT 5.4 xhigh fast)에게
리뷰받고, 피드백을 바탕으로 누락/보완 사항을 반영하여 플랜을 재작성한다.

**핵심**: 플랜은 파일이 아니라 현재 화면에 출력된 상태다.
`cmux read-screen`으로 현재 pane의 내용을 캡처하여 Codex에 전달한다.

## Workflow

### 1. cmux 환경 확인

```bash
if [ -z "$CMUX_WORKSPACE_ID" ]; then
  echo "ERROR: cmux 환경이 아닙니다. cmux 터미널에서 실행해주세요."
  exit 1
fi
```

### 2. 현재 화면에서 플랜 내용 캡처

현재 Superpowers pane에 출력된 플랜 내용을 읽어서 임시 파일에 저장한다.
이 pane이 바로 플랜이 작성된 곳이다.

```bash
# 현재 surface(Superpowers pane)의 화면 내용을 충분히 읽는다
# 플랜이 길 수 있으므로 넉넉하게 500줄 캡처
cmux read-screen --lines 500 > /tmp/current-plan-raw.txt
```

캡처한 내용에서 플랜 부분만 추출하여 정리한다:
- 터미널 프롬프트, 명령어 입력 라인 등 노이즈를 제거
- 실제 플랜 내용(마크다운 헤더, 목록 등)만 `/tmp/current-plan.md`에 저장

```bash
# 노이즈 제거 후 플랜 내용만 저장
# (Claude가 캡처 내용을 읽고 플랜 부분만 추출하여 저장)
```

**중요**: `read-screen`으로 캡처한 텍스트를 직접 읽어서 플랜에 해당하는 부분을 판별한다.
터미널 프롬프트(`$`, `❯` 등), Claude Code UI 요소, 명령어 라인은 제외하고
실제 플랜 본문만 `/tmp/current-plan.md`에 Write 도구로 저장한다.

### 3. 오른쪽에 Codex pane 자동 생성

```bash
cmux new-split right
sleep 1
```

분할 후 새로 생성된 surface reference를 확인한다:
```bash
cmux list-workspaces
```

### 4. Codex에 리뷰 요청 전송

새 pane에 Codex를 실행한다. `/tmp/current-plan.md`(캡처한 플랜)를 읽어서 리뷰하게 한다:

```bash
cmux send --surface <new-surface> "codex --model gpt-5.4-xhigh-fast --quiet 'You are a senior architect reviewer. Read and review the implementation plan in /tmp/current-plan.md thoroughly.\n\nFocus on:\n1. Missing edge cases or error scenarios\n2. Architecture or design concerns\n3. Performance bottlenecks\n4. Security vulnerabilities\n5. Testing gaps\n6. Dependencies or ordering issues\n\nBe specific and actionable. Write your complete review to /tmp/codex-plan-review.md'\n"
```

### 5. 리뷰 완료 대기

사이드바에 진행 상황 표시:
```bash
cmux set-progress 0.3 --label "Codex reviewing plan..."
```

완료 판별 — 다음 두 가지를 반복 확인 (5초 간격):
1. `/tmp/codex-plan-review.md` 파일이 존재하는지
2. `cmux read-screen --surface <new-surface> --lines 10`에서 Codex가 종료되었는지

```bash
cmux set-progress 0.7 --label "Review received, analyzing..."
```

### 6. 리뷰 결과를 Superpowers(현재 pane)에서 읽기

```bash
cat /tmp/codex-plan-review.md
```

리뷰 내용을 전부 읽어온다. 이 내용이 플랜 재작성의 입력이 된다.

### 7. 플랜 재작성 (핵심)

Codex 리뷰를 바탕으로 **현재 플랜을 처음부터 다시 생각한다**:

- 리뷰에서 지적한 **누락된 내용** 추가
- **아키텍처 우려사항** 반영하여 설계 수정
- **엣지 케이스**와 **에러 시나리오** 보강
- **테스트 전략** 보완
- **태스크 순서/의존성** 재정렬

재작성한 플랜을 화면에 다시 출력한다.
단, 상단에 Codex 리뷰 반영 사항을 요약한다:

```
## Codex Review 반영 사항 (GPT 5.4 xhigh fast)
- [누락되어 추가한 항목들]
- [수정/보강한 항목들]
- [Codex가 지적했지만 반영하지 않은 항목과 그 이유]

---
(재작성된 플랜 본문)
```

### 8. 완료 알림

```bash
cmux set-progress 1.0 --label "Plan rewritten with Codex review"
cmux notify --title "Plan Review Complete" --body "Plan has been rewritten based on Codex feedback."
```

## Important Notes

- **플랜은 파일이 아니라 화면 출력이다** — `cmux read-screen`으로 캡처하는 것이 핵심
- Codex pane은 리뷰 후에도 닫지 않는다 (사용자가 직접 확인 가능)
- 캡처 원본: `/tmp/current-plan-raw.txt`, 정리본: `/tmp/current-plan.md`
- 리뷰 원본: `/tmp/codex-plan-review.md`
- 이 스킬은 Superpowers 쪽에서 실행되며, Codex는 리뷰어 역할만 수행한다
