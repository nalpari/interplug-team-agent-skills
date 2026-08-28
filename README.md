# interplug-team-agent-skills

인터플러그 팀에서 공용으로 쓰는 Agent Skills 모음. Claude Code / Codex / Cursor 등
SKILL.md 규격을 지원하는 에이전트에서 사용한다.

## 설치

```bash
npx skills add nalpari/interplug-team-agent-skills --list          # 목록만 보기
npx skills add nalpari/interplug-team-agent-skills@ip-commit-push  # 개별 설치
```

프로젝트가 아니라 전역에 설치하려면 `-g` 를 붙인다.

## 스킬 목록

| 스킬 | 설명 |
|------|------|
| [`ip-commit-push`](ip-commit-push/SKILL.md) | 한글 커밋 메세지 규칙에 맞게 의도 단위로 분할 커밋하고 푸시 |
| [`ip-code-review`](ip-code-review/SKILL.md) | codex + Claude Opus 5 가 동시에 PR 적대적 리뷰, 종합해서 머지 블로커 보고 |
| [`ip-code-review-claude`](ip-code-review-claude/SKILL.md) | codex 없이 Claude 서브 에이전트 3개가 PR 적대적 리뷰, 머지 블로커만 코멘트 |
| [`ip-design-skill`](ip-design-skill/SKILL.md) | 디자인 스킬 4종을 방향→다이얼→이미지→구현→감사 순서로 물려서 한국어 화면 제작 (Claude Code 전용) |

## 새 스킬 추가하기

### 1. 디렉토리와 파일

```
ip-<이름>/SKILL.md
```

- **`ip-` prefix 필수.** skills.sh 레지스트리에 `commit-push`, `git-commit-push` 같은
  일반적인 이름이 이미 많아 묻힌다. 팀 prefix로 구분하고 레포 내 이름 규칙도 통일한다.
- 디렉토리명과 frontmatter의 `name` 은 반드시 같아야 한다.
- 스크립트나 참조 문서가 꼭 필요한 경우가 아니면 `SKILL.md` 한 파일로 끝낸다.

### 2. frontmatter

```yaml
---
name: ip-example
description: '무엇을 하는 스킬인지 한 문장. "이렇게 말하면", "이런 요청일 때" 사용한다.'
---
```

- `description` 이 트리거 조건이다. 기능 설명만 쓰지 말고 **사용자가 실제로 쓸 말**을
  따옴표로 넣어라. 한국어와 영어 표현을 함께 넣으면 매칭이 잘 된다.
- 검색 노출을 위해 차별점이 되는 키워드를 앞쪽에 배치한다.

**YAML 주의사항** — 아래 문자가 들어가면 값 전체를 싱글쿼트로 감싼다:

| 위치 | 문자 | 결과 |
|------|------|------|
| 값 어디든 | `: ` (콜론+공백) | 중첩 매핑으로 오해 → 파싱 실패 |
| 값 맨 앞 | `{ [ & * ! \| > % @ ` ` " '` | 특수 노드로 오해 → 파싱 실패 |

파싱이 깨지면 skills.sh 가 스킬을 조용히 건너뛴다 (`No skills found`).

### 3. 검증

커밋 전에 frontmatter가 파싱되는지 확인한다.

```bash
awk 'NR>1 && /^---$/{exit} NR>1' ip-example/SKILL.md | npx -y js-yaml /dev/stdin
```

`name` 과 `description` 이 출력되면 통과. 레포 전체 검사:

```bash
for f in */SKILL.md; do
  awk 'NR>1 && /^---$/{exit} NR>1' "$f" | npx -y js-yaml /dev/stdin > /dev/null \
    && echo "OK   $f" || echo "FAIL $f"
done
```

### 4. 내용 작성

- 절차는 번호 목록으로, 판단 기준은 표나 불릿으로 쓴다.
- 에이전트가 **쓸 수 없는 것**은 명시한다 (e.g. `git add -p` 는 인터랙티브라 불가).
  안 적으면 에이전트가 시도하다 막힌다.
- 설명이 규칙보다 길어지면 규칙이 아니라 산문이다. 줄여라.

### 5. 커밋

커밋 메세지 규칙은 [`ip-commit-push`](ip-commit-push/SKILL.md) 를 따른다.
스킬을 설치해뒀다면 `커밋하고 푸시해줘` 로 위임하면 된다.

### 6. skills.sh 검색 등록

`--list` 는 GitHub를 직접 clone하므로 레지스트리를 거치지 않는다. 검색에 노출되려면
**실제 설치 이벤트**가 한 번은 필요하다.

```bash
npx skills add nalpari/interplug-team-agent-skills@ip-example -g -y
```

설치 직후 상세 페이지(`https://skills.sh/nalpari/interplug-team-agent-skills/<스킬명>`)가
생성되고, 검색/리더보드 인덱스에는 시간이 걸린다. 순위는 설치수 기준이다.

### 로컬에서 개발할 때

`skills add` 는 파일을 **복사**하므로 레포를 수정해도 반영되지 않는다. 개발 중에는
심볼릭 링크를 쓴다.

```bash
ln -s "$PWD/ip-example" ~/.claude/skills/ip-example
```
