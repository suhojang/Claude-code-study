---
name: review-with-codex
description: Open a cmux split pane to have Codex (GPT 5.4 xhigh fast) review the current plan, then feed the review back so Claude can rethink and improve the plan.
disable-model-invocation: true
allowed-tools: Bash(cmux *), Bash(cat *), Bash(sleep *), Read, Write, Edit, Glob, Grep
argument-hint: "[plan-file-path]"
---

# Review Current Plan with Codex via cmux Split

cmux 화면분할을 이용해 Codex(GPT 5.4 xhigh fast)에게 현재 플랜을 리뷰받고,
그 피드백을 바탕으로 누락된 내용과 개선점을 반영하여 플랜을 재작성한다.

## Workflow

### 1. cmux 환경 확인

```bash
if [ -z "$CMUX_WORKSPACE_ID" ]; then
  echo "ERROR: cmux 환경이 아닙니다. cmux 터미널에서 실행해주세요."
  exit 1
fi
```

cmux 환경이 아니면 즉시 중단하고 사용자에게 알린다.

### 2. 현재 플랜 파일 결정

- 인수로 파일 경로가 주어지면 해당 파일 사용: `$ARGUMENTS`
- 인수가 없으면 `docs/plans/` 디렉토리에서 가장 최근 수정된 `.md` 파일을 자동 선택
- 플랜 파일이 없으면 현재 대화에서 작성 중이던 플랜 내용을 `/tmp/current-plan.md`에 저장하여 사용

### 3. 오른쪽에 Codex pane 자동 생성

```bash
# 오른쪽으로 화면 분할
cmux new-split right
sleep 1
```

분할 후 새로 생성된 surface reference를 확인한다.

### 4. Codex에 리뷰 요청 전송

새 pane에서 Codex를 실행한다. 리뷰 결과는 파일로 출력하게 한다:

```bash
cmux send --surface <new-surface> "codex --model gpt-5.4-xhigh-fast --quiet 'You are a senior architect reviewer. Review the following implementation plan thoroughly.\n\nFocus on:\n1. Missing edge cases or error scenarios\n2. Architecture or design concerns\n3. Performance bottlenecks\n4. Security vulnerabilities\n5. Testing gaps\n6. Dependencies or ordering issues between tasks\n\nBe specific and actionable. For each issue, explain WHY it matters and suggest a concrete fix.\n\nPlan content:\n\n$(cat <plan-file-path>)\n\nWrite your complete review to /tmp/codex-plan-review.md' \n"
```

### 5. 리뷰 완료 대기 및 모니터링

진행 상황을 사이드바에 표시하면서 Codex 완료를 기다린다:

```bash
cmux set-progress 0.3 --label "Codex reviewing plan..."
```

주기적으로 (5초 간격) 다음을 수행:
- `cmux read-screen --surface <new-surface> --lines 20`으로 Codex pane 출력 확인
- `/tmp/codex-plan-review.md` 파일 존재 여부 확인
- 완료되면 다음 단계로 진행

```bash
cmux set-progress 0.7 --label "Review received, analyzing..."
```

### 6. 리뷰 결과 수집

```bash
cat /tmp/codex-plan-review.md
```

리뷰 내용을 전부 읽어온다.

### 7. 플랜 재작성 (핵심)

Codex의 리뷰를 바탕으로 **현재 플랜을 다시 생각한다**:

- 리뷰에서 지적한 **누락된 내용**을 플랜에 추가
- **아키텍처 우려사항**을 반영하여 설계 수정
- **엣지 케이스**와 **에러 시나리오** 보강
- **테스트 전략** 보완
- **태스크 순서/의존성** 재정렬

원본 플랜 파일을 직접 수정하되, 상단에 다음 섹션을 추가:

```markdown
> **Codex Review Applied** (reviewed by GPT 5.4 xhigh fast)
> - [반영한 주요 피드백 요약]
> - [추가된 항목들]
> - [수정된 항목들]
```

### 8. 완료 알림

```bash
cmux set-progress 1.0 --label "Plan updated with Codex review"
cmux notify --title "Plan Review Complete" --body "Codex review applied. Plan has been improved."
```

## Important Notes

- Codex pane은 리뷰가 끝나도 닫지 않는다 (사용자가 직접 확인할 수 있도록)
- 리뷰 원본은 `/tmp/codex-plan-review.md`에 보존된다
- 플랜 재작성 시 기존 내용을 삭제하지 않고, 보강/수정한다
- 이 스킬은 Superpowers(Claude) 쪽에서 실행되며, Codex는 리뷰어 역할만 수행한다
