---
name: ip-code-review
description: 'PR 번호와 브랜치를 받아 해당 브랜치로 체크아웃한 뒤 codex adversarial-review 와 Claude Opus 5 서브에이전트가 동시에 적대적 코드리뷰를 수행하고, lead 가 두 결과를 종합해 머지 블로커 중심으로 보고한다. "PR 리뷰해줘", "머지해도 되는지 봐줘", "적대적 코드리뷰", "codex랑 claude 둘 다 리뷰해줘", "머지 전에 문제 없는지 확인", "multi-model PR review" 처럼 머지 전 PR 검증을 요청할 때 사용한다.'
---

# Dual-Engine Adversarial PR Review

서로 다른 두 엔진이 같은 PR을 독립적으로 적대적 리뷰하고, lead 가 종합해 **머지 블로커**를 뽑아낸다.

| 역할 | 주체 | 하는 일 |
|------|------|---------|
| 리뷰어 A | codex `adversarial-review` | 구현 방식·설계 선택·가정을 공격 |
| 리뷰어 B | Claude Opus 5 서브에이전트 | 같은 PR을 적대적 관점으로 리뷰 |
| lead | 이 세션 (Opus 5) | 체크아웃, 두 리뷰 동시 실행, 종합, 보고 |

lead 는 **직접 리뷰하지 않는다.** 조율과 종합만 한다.

## 입력

- **브랜치** — 체크아웃할 브랜치. 필수.
- **PR 번호** — 리뷰 대상 PR. 필수.

둘 중 하나라도 없으면 물어본다. 추측하지 않는다.

## 절차

### 1. 체크아웃

체크아웃 전에 워킹 트리를 확인한다.

```bash
git status --short
```

- **커밋 안 된 변경이 있으면 멈추고 사용자에게 알린다.** 체크아웃은 남의 작업을 날릴 수 있다. stash 를 임의로 하지 않는다.
- 깨끗하면 진행한다.

```bash
git fetch origin
git checkout <브랜치>
git pull --ff-only
```

리뷰가 끝나도 원래 브랜치로 되돌리지 않는다. 보통 이어서 수정하게 된다.

### 2. PR 메타 확인

```bash
gh pr view <n> --json number,title,url,baseRefName,headRefName,additions,deletions,changedFiles
```

- `headRefName` 이 방금 체크아웃한 브랜치와 다르면 **멈추고 사용자에게 알린다.** 엉뚱한 코드를 리뷰하게 된다.
- `baseRefName` 을 base ref 로 쓴다. 아래에서 `origin/<baseRefName>` 형태로 넘긴다.

### 3. 두 리뷰를 동시에 실행

**순차로 돌리지 않는다.** codex 를 백그라운드로 먼저 띄우고, 곧바로 서브에이전트를 호출한다.

#### 3-1. codex adversarial-review (백그라운드)

`/codex:adversarial-review` 슬래시 커맨드는 `disable-model-invocation` 이라 에이전트가 호출할 수 없다. 그 커맨드가 실행하는 companion 스크립트를 직접 돌린다.

```bash
CODEX_ROOT=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
node "${CODEX_ROOT}scripts/codex-companion.mjs" adversarial-review --wait --base origin/<base> --scope branch
```

- Bash 도구의 `run_in_background: true` 로 실행한다. 스크립트의 `--wait` 는 그대로 두고 detach 는 Bash 쪽에서 한다.
- `CODEX_ROOT` 가 안 잡히면 codex 플러그인이 없는 것이다. 임의로 다른 경로를 찾지 말고 사용자에게 알린 뒤, Claude 리뷰만 단독으로 진행할지 묻는다.
- 이 스크립트는 PR 번호를 받지 않는다. 체크아웃 + `--base` 로 대상이 정해지므로 1·2단계 순서가 중요하다.

#### 3-2. Claude Opus 5 서브에이전트

codex 를 띄운 직후 호출한다.

```
Agent(
  subagent_type: "general-purpose",
  model: "opus",
  description: "PR 적대적 리뷰",
  prompt: "<아래 프롬프트>"
)
```

**codex 결과를 서브에이전트에 알려주지 않는다.** 두 리뷰는 서로를 모른 채 독립적이어야 교차 검증이 된다.

#### 3-3. 수거

서브에이전트가 반환되면 `BashOutput` 으로 codex 결과를 가져온다. 아직 안 끝났으면 기다린다. codex 리뷰는 수 분 걸릴 수 있고, 출력이 없다고 실패가 아니다.

### 4. lead 가 종합해 보고

아래 "보고 형식"대로 쓴다.

## 서브에이전트 프롬프트

`<n>`, `<base>`, `<branch>` 를 실제 값으로 치환한다.

```
PR #<n> (<branch> -> <base>) 를 적대적 시각으로 코드리뷰한다. 승인이 목적이 아니라 머지하면 터질 것을 찾는 것이 목적이다.

대상 diff:
  git diff origin/<base>...HEAD
필요하면 주변 파일을 직접 읽어 맥락을 확인한다. diff 만 보고 판단하지 않는다.

규칙:
- diff 에 실제로 있는 코드만 지적한다. 없는 코드를 추측으로 만들지 않는다.
- 지적마다 `파일:줄`, 무엇이 잘못됐는지, 그것이 실제로 터지는 구체적 시나리오(어떤 입력/상태 -> 어떤 잘못된 결과)를 쓴다. 시나리오를 못 쓰겠으면 그 지적은 버린다.
- severity 를 blocker / major / minor 로 붙인다.
  - blocker = 이대로 머지하면 프로덕션에서 실패하거나 데이터/보안이 깨진다.
  - major = 고쳐야 하지만 머지를 막을 정도는 아니다.
  - minor = 있으면 좋은 개선.
- 취향 문제(네이밍, 포맷, 스타일)는 쓰지 않는다. 동작 오류 / 보안 / 데이터 정합성 / 동시성 / 에러 처리 / 성능 / 테스트 공백에 집중한다.
- 기존 코드의 문제는 이 PR 이 건드렸거나 이 PR 때문에 터지는 경우에만 쓴다.

파일을 수정하지 않는다. 리뷰 결과만 반환한다.

blocker 를 맨 위에 두고 severity 순으로 정리해 반환한다. blocker 가 없으면 없다고 명시한다.
```

## 보고 형식

두 리뷰를 합쳐 lead 가 재작성한다. 양쪽 출력을 그대로 이어붙이지 않는다.

- **머지 블로커** — 최우선. 고치기 전에는 머지하면 안 되는 것만. 각 항목에 `파일:줄`, 실패 시나리오, 어느 엔진이 잡았는지(codex / claude / 양쪽). **양쪽이 독립적으로 같은 것을 지적했으면 표시한다** — 신뢰도가 가장 높은 신호다.
- **이견** — 한쪽만 지적했고 lead 가 보기에 판단이 갈리는 것. 양쪽 논지를 한 줄씩 쓰고 lead 의 판단을 붙인다.
- **머지 후 처리 가능** — major / minor. 한 줄씩, 짧게.
- **기각** — 근거가 시나리오까지 가지 않거나 취향 문제인 지적. 한 줄씩.

마지막에 **머지 가부 판단**을 한 문장으로 쓴다. blocker 가 0이면 그렇게 말한다.

**lead 는 파일을 고치지 않는다.** 이 스킬의 산출물은 리뷰 결과다. 수정은 사용자가 따로 요청할 때 진행한다.

## 하지 말 것

- 더러운 워킹 트리에서 체크아웃 강행하기. 멈추고 알린다.
- lead 가 직접 리뷰하기. 세 번째 의견이 아니라 종합이 lead 의 일이다.
- codex 결과를 서브에이전트에 흘리기. 독립성이 이 스킬의 전부다.
- 두 리뷰를 순차로 돌리기. 백그라운드 + 서브에이전트로 동시에 돌린다.
- codex 가 조용하다고 실패로 판정하고 넘어가기. 수 분 걸린다.
- blocker 를 minor 와 같은 비중으로 나열하기. 머지 블로커가 이 보고서의 목적이다.
