# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 이 레포의 성격

실행되는 코드가 없다. 빌드·린트·테스트 스크립트도, 의존성도 없다. 산출물은
`<스킬명>/SKILL.md` 텍스트 하나뿐이고, 이 파일을 소비하는 주체는 사람이 아니라
에이전트다. 따라서 "동작한다"의 의미가 일반 코드베이스와 다르다.

## 검증

유일한 자동 검증은 frontmatter가 YAML로 파싱되는지 확인하는 것이다. 파싱이 깨지면
skills.sh 와 에이전트가 스킬을 **조용히 건너뛴다** — 에러 없이 그냥 없는 스킬이 된다.
SKILL.md 를 수정했으면 반드시 실행한다.

```bash
# 단일 스킬
awk 'NR>1 && /^---$/{exit} NR>1' <스킬명>/SKILL.md | npx -y js-yaml /dev/stdin

# 전체
for f in */SKILL.md; do
  awk 'NR>1 && /^---$/{exit} NR>1' "$f" | npx -y js-yaml /dev/stdin > /dev/null \
    && echo "OK   $f" || echo "FAIL $f"
done
```

파싱 함정과 회피 방법은 `README.md` 의 "새 스킬 추가하기 > 2. frontmatter" 참고.

## 편집 시 주의

- **`~/.claude/skills/<스킬명>` 이 이 레포로 심볼릭 링크되어 있을 수 있다.** 그 경우
  여기서의 편집이 활성 스킬에 즉시 반영된다. 실험적인 수정을 커밋 전 상태로 방치하지 말 것.
- `name` 은 디렉토리명과 반드시 일치해야 한다. 디렉토리를 옮기면 `name` 도 같이 고친다.
- `description` 은 문서가 아니라 **트리거 조건**이다. 여기를 고치는 것은 스킬이 언제
  발동하는지를 바꾸는 것이므로 `feat`/`fix` 로 커밋한다. 본문 문구만 다듬은 경우는 `docs`.

## 커밋

커밋 메세지 규칙은 이 레포가 담고 있는 `ip-commit-push/SKILL.md` 를 그대로 따른다
(self-hosting). 규칙을 바꾸려면 그 파일을 고치는 것이 곧 규칙 변경이다.

## 새 스킬 추가

절차·이름 규칙(`ip-` prefix)·skills.sh 등록 방법은 `README.md` 가 유일한 기준이다.
여기에 중복 기술하지 말고 README 를 갱신한다.
