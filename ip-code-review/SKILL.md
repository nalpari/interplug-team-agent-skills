---
name: ip-code-review
description: 'Orca orchestration으로 Claude Opus 5 / Fable 5 와 GPT 5.6 sol / terra / luna 워커 5개를 띄워 PR을 적대적으로 코드리뷰하고 상호 반박 토론까지 시킨 뒤 lead가 종합 보고한다. "멀티 에이전트 코드리뷰", "적대적 코드리뷰", "5개 모델로 PR 리뷰", "워커 띄워서 리뷰해줘", "adversarial code review", "multi-agent PR review" 처럼 여러 모델을 붙여 PR을 검증하려는 요청에 사용한다. 단일 모델 리뷰나 단순 diff 확인에는 쓰지 않는다.'
---

# Adversarial Multi-Agent Code Review

lead 1 + 워커 5로 PR을 적대적으로 리뷰하고, 워커끼리 서로의 발견을 반박시킨 뒤 lead가 종합한다.

## 전제

- 이 스킬을 실행하는 세션(lead)이 **Opus 5** 여야 한다. lead는 조율과 종합만 하고 직접 리뷰하지 않는다.
- Orca 런타임이 떠 있어야 하고(`orca status --json`), Settings > Experimental 에서 orchestration 이 켜져 있어야 한다.
- **시작 전 반드시 `orca skills get orchestration` 으로 버전에 맞는 가이드를 읽는다.** 아래 명령은 요약이고 플래그는 릴리즈마다 바뀐다.
- 실행 파일은 orchestration 스킬의 "Resolve the CLI" 규칙대로 고른다 (`$ORCA_CLI_COMMAND` / `orca-dev` / `orca-ide` / `orca`). 아래에선 `orca` 로 표기한다.

## 워커 구성

| 워커 | `--agent` | `--model` |
|------|-----------|-----------|
| opus | `claude` | `claude-opus-5` |
| fable | `claude` | `claude-fable-5` |
| sol | `codex` | `gpt-5.6-sol` |
| terra | `codex` | `gpt-5.6-terra` |
| luna | `codex` | `gpt-5.6-luna` |

- 전원 `--worktree current`. 리뷰는 읽기 전용이라 새 worktree 를 만들 이유가 없다.
- `--effort` 는 붙이지 않는다(모델 기본값). 사용자가 명시하면 `--model` 과 함께만 쓴다.
- 모델 id 가 거부되면 **그 워커만 빼고 진행**하고 보고에 적는다. 기억에 있는 다른 id 로 바꾸지 않는다.
- 관점 배분(보안 담당, 성능 담당 …)은 하지 않는다. 모델이 다른 것 자체가 다양성이고, 같은 프롬프트여야 모델 간 비교가 된다.

## 절차

### 1. 리뷰 대상 확정

- 인자에 PR 번호가 있으면 그것, 없으면 현재 브랜치의 PR.
- `gh pr view <n> --json title,body,baseRefName,headRefName` 로 base/head 를 확인한다.
- PR 이 없으면 base 대비 diff (`git diff origin/main...HEAD`) 로 대체하고 사용자에게 알린다.
- **diff 본문을 프롬프트에 붙여넣지 않는다.** 워커가 같은 worktree 에 있으므로 명령만 알려주면 직접 읽는다.

### 2. Run + 라운드 1 Task 5개 생성

```bash
orca orchestration run-create --objective "PR #<n> 적대적 멀티모델 코드리뷰" --json
orca orchestration task-create --spec "<라운드 1 프롬프트>" --json   # 워커 수만큼, 총 5회
```

### 3. 워커 5개를 전부 띄운 뒤에 기다린다

```bash
orca orchestration worker-start --task <task_id> --worktree current --agent claude --model claude-opus-5 --json
# … 표대로 5개
```

receipt 의 `dispatch` id 와 `worker.agent_terminal_handle` 을 워커 이름과 짝지어 기록한다. 라운드 2와 정리 단계에서 쓴다.

### 4. 라운드 1 수집

```bash
orca orchestration check --wait --types worker_done,escalation,question --timeout-ms 900000 --json
```

- 타임아웃이나 `count:0` 은 실패가 아니라 체크포인트다. 5개가 전부 settle 할 때까지 rolling wait 한다.
- `question` 은 `orca orchestration reply --id <msg_id> --body <답> --json` 으로 답한다.
- Delivery 안의 메세지를 **전부 처리한 뒤에만** `--ack <delivery_id>`.
- **여기서 `worker-release` 하지 않는다.** 라운드 2에서 같은 터미널을 재사용한다.

### 5. 라운드 2 (토론)

Task 5개를 새로 만들고 같은 터미널에 재배정한다.

```bash
orca orchestration task-create --spec "<라운드 2 프롬프트>" --json
orca orchestration worker-start --task <task2_id> --terminal <그 워커의 handle> --json
```

`--terminal` 은 `--agent` / `--model` 과 같이 못 쓴다. 핸들만 준다.

### 6. 라운드 2 수집

4번과 동일. 라운드는 2회로 끝낸다. 3라운드는 이견이 핵심 이슈에 남았고 사용자가 요청할 때만.

### 7. 정리

```bash
orca orchestration worker-release --dispatch <dispatch_id> --json   # 워커마다
```

사용자가 터미널 유지를 원하면 `worker-release` 대신 `worker-retain`.

### 8. lead 보고

아래 "보고 형식"대로 사용자에게 보고한다.

## 라운드 1 프롬프트

5개 워커에 동일하게 준다. `<n>`, `<base>`, `<head>` 는 실제 값으로 치환한다.

```
PR #<n> (<head> -> <base>) 를 적대적 시각으로 코드리뷰한다. 승인이 목적이 아니라 결함을 찾는 것이 목적이다.

대상: `gh pr diff <n>` (PR 이 없으면 `git diff <base>...<head>`). 필요하면 주변 파일을 직접 읽어 맥락을 확인한다.

규칙:
- diff 에 실제로 있는 코드만 지적한다. 없는 코드를 추측으로 만들지 않는다.
- 지적마다 `파일:줄`, 무엇이 잘못됐는지, 그리고 그것이 실제로 터지는 구체적 시나리오(어떤 입력/상태 -> 어떤 잘못된 결과)를 쓴다. 시나리오를 못 쓰겠으면 그 지적은 버린다.
- 취향 문제(네이밍, 포맷, 스타일)는 쓰지 않는다. 동작 오류 / 보안 / 데이터 정합성 / 성능 / 에러 처리 / 테스트 공백에 집중한다.
- severity 를 critical / major / minor 로 붙인다.

파일을 수정하지 않는다. 다른 워커와 같은 worktree 를 공유하므로 편집은 충돌을 만든다. 읽기와 보고만 한다.

발견을 severity 순으로 정리해 worker_done 으로 보고한다. 발견이 없으면 없다고 보고한다.
```

## 라운드 2 프롬프트

lead 가 다른 4명의 라운드 1 결과를 채워 넣는다. 본인 것은 빼고 준다.

```
아래는 같은 PR 에 대한 다른 워커 4명의 리뷰 결과다.

[<워커명>]
<발견 목록>
… 4명분

너의 라운드 1 결과는 네가 이미 알고 있다. 이제:
1. 남들의 지적마다 동의 / 반박 / 보류 중 하나를 고르고 근거를 쓴다. 반박은 "왜 그 시나리오가 실제로 발생하지 않는지"를 코드로 보여야 한다. 근거 없는 동의는 하지 않는다.
2. 남들이 지적했고 네가 놓친 것 중 실제 결함인 것은 인정한다.
3. 남들의 지적을 읽다가 새로 발견한 것이 있으면 추가한다.

역시 파일을 수정하지 않는다. 결과를 worker_done 으로 보고한다.
```

## 보고 형식

두 라운드를 합쳐 lead 가 재작성한다. 워커 발언을 그대로 붙여넣지 않는다.

- **확정 이슈** — 2명 이상이 지적했고 라운드 2에서 반박되지 않은 것. severity 순으로 `파일:줄` + 시나리오 + 지적한 워커.
- **이견 이슈** — 지적과 반박이 갈린 것. 양쪽 근거를 한 줄씩 쓰고 lead 의 판단을 붙인다.
- **기각 이슈** — 라운드 2에서 반박된 것. 한 줄씩.
- **모델별 특이점** — 특정 모델만 잡아낸 게 있으면 짧게. 없으면 생략.

끝에 조치 제안을 붙인다. **lead 는 파일을 고치지 않는다.** 이 스킬의 산출물은 리뷰 결과이고, 수정은 사용자가 따로 요청할 때 진행한다.

## 하지 말 것

- 워커 대신 lead 가 리뷰하기. lead 는 조율과 종합만 한다.
- 워커를 하나 띄우고 기다렸다가 다음을 띄우기. Task 를 전부 만들고 `worker-start` 를 전부 실행한 뒤에 wait 한다.
- 안 끝났다고 워커를 stop / close / kill 하기. heartbeat 와 터미널 활동은 살아있다는 뜻이지 멈춘 게 아니다.
- 새 worktree 만들기. 리뷰는 읽기 전용이라 격리가 필요 없다.
- Orca 대신 다른 subagent 도구로 워커 만들기. task/dispatch 기록과 `worker_done` 권한이 안 생긴다.
