---
name: ip-design-skill
description: '디자인 스킬 4종(frontend-design, design-taste-frontend, image-to-code, impeccable)을 방향→다이얼→이미지→구현→감사 순서로 물려서 한국어 화면을 설계·구현·감사한다. "디자인해줘", "화면 만들어줘", "UI 만들어줘", "랜딩페이지 만들어줘", "대시보드 화면 짜줘", "design this page", "build this UI" 처럼 화면·UI 제작을 요청할 때 사용한다. 빠진 스킬은 사용자 승인을 받은 뒤 설치한다. Claude Code 전용.'
---

# 디자인 4종 세트로 만든다

사용자가 매번
*"design-taste-frontend, image-to-code, impeccable, frontend-design 적극 활용해서 디자인해줘"*
라고 길게 치는 걸 없애는 것이 이 스킬의 목적이다. 그러니 **묻지 말고 바로 시작한다.**
무엇을 만들지 알 수 없을 때만 한 줄로 물어본다.

## 0단계: 준비 — Claude Code 전용

이 단계의 확인 경로와 설치 경로는 **Claude Code 기준(`.claude/skills/`)이다.**
Codex·Cursor 에서 실행 중이면 0단계를 건너뛰고, 4종이 이미 설치돼 있는지
**사용자에게 먼저 확인한 뒤** 1단계로 간다. 확인 없이 진행하면 4종 없이 만든 결과를
4종으로 만든 결과로 오인하게 된다.

4종 중 3개(`design-taste-frontend`, `image-to-code`, `impeccable`)만 이 루프로 확인한다.
`frontend-design` 은 스킬이 아니라 마켓플레이스 플러그인이라 `.claude/skills/` 가 아닌
`~/.claude/plugins/` 밑에 깔린다. 여기서는 잡히지 않으므로 1단계에서 직접 확인한다.

```bash
NEED=""
for s in design-taste-frontend image-to-code impeccable; do
  state=missing
  for base in ".claude/skills" "$HOME/.claude/skills"; do
    if   [ -f "$base/$s/SKILL.md" ]; then state=ok; break
    elif [ -e "$base/$s" ];          then state="conflict:$base/$s"; break
    fi
  done
  case "$state" in
    ok)      ;;
    missing) NEED="$NEED $s" ;;
    *)       echo "CONFLICT:$s -> ${state#conflict:}" ;;
  esac
done
echo "NEED:$NEED"
```

`NEED` 가 비어 있고 `CONFLICT` 도 없으면 **이 단계를 통째로 건너뛰고 1단계로 간다.**
보고도 하지 않는다. 준비됐다는 말은 사용자가 원한 결과물이 아니다.

**`CONFLICT` 가 하나라도 나오면 거기서 멈추고 사용자에게 알린다.** 그 이름으로 뭔가가
이미 있는데 `SKILL.md` 가 없는 상태다. 작업 중인 자체 스킬이거나, 중단된 설치이거나,
이름 충돌이다. 설치로 넘어가면 안 된다. `skills add --copy` 는 복사 전에 **목적지를
통째로 지우고 다시 만든다** (`cleanAndCreateDirectory`). 그 안의 사용자 파일은 복구할 수 없다.
디렉토리가 아니라 같은 이름의 **파일**이 있어도 마찬가지라서 `-d` 가 아니라 `-e` 로 본다.
(끊어진 심볼릭 링크는 `-e` 도 false 라 `missing` 으로 간다. 잃는 게 죽은 링크뿐이라 그대로 둔다.)
사용자가 그것을 치우기 전에는 해당 스킬을 설치하지 않는다.

### 빠진 게 있어도 자동으로 설치하지 않는다

서드파티 레포의 SKILL.md 는 에이전트가 그대로 따르는 지시문이다. 코드 의존성과 같은
무게로 다룬다. **설치 명령을 보여주고 사용자 승인을 받은 뒤에만 실행한다.**
"화면 만들어줘" 한 마디가 외부 지시문 설치 승인이 될 수는 없다.

승인을 요청할 때 네 가지를 같이 적는다:

- 무엇이 빠졌는지 — `NEED` 에 담긴 것만
- 어디서 받는지 — 레포 주소와 아래의 **고정 커밋 SHA**
- 어디에 깔리는지 — 프로젝트의 `.claude/skills/`. 전역이 아니라 **사용자의 워킹트리**다
- **무엇이 들어오는지** — 프롬프트 문서만이 아니다. `impeccable` 만 해도 `.mjs` 실행
  스크립트 100여 개와 훅 런타임, 서브에이전트 정의가 함께 깔린다. 전부 에이전트 권한으로
  실행된다. `.gitignore` 에 넣을지도 같이 물어본다

승인받으면 아래를 실행한다. **`APPROVED` 에 사용자가 방금 승인한 이름만 적는다.**
`NEED` 를 셸 변수로 넘겨받으려 하지 마라. 승인 문답이 끼면 셸이 바뀌어 값이 사라지고,
확인 루프를 다시 돌리면 **거절당한 스킬까지 되살아난다.**

```bash
set -euo pipefail
APPROVED=""   # 사용자가 방금 승인한 이름만 여기 적는다. 비어 있으면 아무것도 설치하지 않는다

bak=$(mktemp ./skills-lock.json.bak.XXXXXX); had=0   # 락파일 옆에, 고유 이름으로
mkdir .skills-install.lock 2>/dev/null \
  || { echo "ABORT: 이 디렉토리에서 다른 설치가 진행 중이다"; rm -f "$bak"; exit 1; }
if [ -e skills-lock.json ]; then cp skills-lock.json "$bak"; had=1; fi
trap 'if [ "$had" = 1 ]; then mv -f "$bak" skills-lock.json; else rm -f skills-lock.json "$bak"; fi
      rmdir .skills-install.lock' EXIT

src() { case "$1" in
  design-taste-frontend|image-to-code)
    echo "https://github.com/Leonxlnx/taste-skill ccbc15639c97057cbfcf32ecebc38ef716e4bb37" ;;
  impeccable)
    echo "https://github.com/pbakaus/impeccable 63b04e2530f5c7b41ea83c133daab24f34912456" ;;  # skill-v4.1.2
esac; }

for s in $APPROVED; do
  state=missing                                    # 설치 직전 다시 판정한다. 덮어쓰기 최후 방어선
  for base in ".claude/skills" "$HOME/.claude/skills"; do
    if   [ -f "$base/$s/SKILL.md" ]; then state=ok; break
    elif [ -e "$base/$s" ];          then state="conflict:$base/$s"; break
    fi
  done
  case "$state" in
    ok)      echo "SKIP:$s 이미 설치돼 있다"; continue ;;
    missing) ;;
    *)       echo "ABORT:$s -> ${state#conflict:} 가 이미 있다"; exit 1 ;;
  esac

  set -- $(src "$s"); repo=$1; sha=$2
  d=$(mktemp -d)                                   # 고정 경로 금지. /tmp 는 누구나 쓴다
  git clone -q --filter=blob:none "$repo" "$d"
  git -C "$d" checkout -q "$sha"
  [ "$(git -C "$d" rev-parse HEAD)" = "$sha" ] && [ -z "$(git -C "$d" status --porcelain)" ] \
    || { echo "FAIL:$s 고정 검증 실패"; rm -rf "$d"; exit 1; }
  npx --yes skills@1.5.23 add "$d" --skill "$s" --agent claude-code --yes --copy
  rm -rf "$d"
done
```

두 코드 블록은 **프로젝트 루트에서** 돌린다. `.claude/skills` 를 상대 경로로 보기 때문에
CWD 가 다르면 확인한 곳과 설치한 곳이 달라진다.

이 스크립트가 지키는 것들이다. 손으로 명령을 쪼개 실행하면서 빼먹지 않는다:

- **`APPROVED` 에 넣어도 이미 있으면 설치하지 않는다.** 설치 직전에 다시 판정해서
  `ok` 면 건너뛰고 `conflict` 면 멈춘다. `--yes --copy` 가 사용자 파일을 지우는 걸
  막는 마지막 지점이다
- **`mktemp -d` 로 매번 새 디렉토리에 받는다.** `/tmp/taste-skill` 같은 고정 경로는
  두 번째 실행에서 `already exists` 로 clone 이 실패하고, 그 자리에 남아 있던
  낡거나 남이 심어둔 트리가 대신 설치된다. 커밋 고정이 조용히 무효가 되는 경로다
- **`set -euo pipefail`** — clone 이나 checkout 이 실패하면 즉시 멈춘다. 다음 줄로 넘어가지 않는다
- **설치 전에 SHA 와 워킹트리를 확인한다.** `rev-parse HEAD` 가 기대한 SHA 와 같고
  `status --porcelain` 이 비어 있어야 설치로 넘어간다
- **설치 후 받은 디렉토리를 지운다**
- **`skills-lock.json` 을 원래대로 되돌린다.** 로컬 경로로 설치하면 CLI 가 이 락파일에
  항목을 추가하는데, 거기 적히는 source 는 레포+SHA 가 아니라 **방금 지운 temp 경로**다
  (`"source": "../tmp.868MM81Ota", "sourceType": "local"`). 커밋되면 그 경로 이름이 공개되고,
  누가 `skills experimental_install` 로 복원하면 SHA 검증도 승인도 없이 그 경로의 내용이 깔린다.
  그렇다고 파일을 지우면 안 된다. 이 락파일은 **프로젝트 전체 레지스트리**라서 CLI 가
  기존 항목을 병합해 쓴다. 지우면 팀이 등록해둔 다른 스킬까지 사라진다.
  그래서 설치 전에 백업하고 `trap ... EXIT` 로 원본을 되돌린다. 원래 없었으면 지운다.
  중간에 `ABORT` 나 `FAIL` 로 죽어도, `Ctrl-C` 로 끊어도 실행되는 자리다.
  백업은 락파일 옆에 `mktemp` 로 고유 이름을 뽑고, 복원 여부는 파일 존재가 아니라
  `had` 플래그로 판단한다. 이름만 고유해서는 부족하다. 같은 디렉토리에서 둘이 동시에 돌면
  나중 실행이 **앞선 실행이 이미 오염시킨 락파일을 "원본" 으로 캡처**해서 되돌려 놓는다.
  둘 다 성공으로 끝나고 temp 경로가 영구히 남는다. 그래서 `mkdir` 뮤텍스로 감싼다.
  고정 이름을 쓰면 남의 파일을 덮거나, 앞선 실행이 남긴 고아 파일을 "원본" 으로 복원한다.
  락파일과 같은 디렉토리에 두는 이유는 두 가지다. `mv` 가 파일시스템을 넘지 않고,
  강제 종료 후에 사람이 찾을 수 있다.
  `kill -9` 로 trap 조차 못 돌았으면 다시 돌리지 말고 이 순서로 되돌린다:
  `skills-lock.json.bak.*` 가 남아 있으면 그걸 `skills-lock.json` 으로 되돌린다.
  백업도 없고 git 이 추적하지도 않으면 `rm -f skills-lock.json` 이다 (원래 없던 파일이다).
  `git checkout --` 는 추적 중이면서 백업도 없을 때만 쓴다.
  커밋 안 한 정상 편집이 같이 날아간다. **락파일로 복원하지 않는다.**
  재설치는 이 스크립트를 다시 돌린다

CLI 로 바로 받지 않고 clone 을 거치는 이유가 있다. `skills add <repo>@<ref>` 는 내부적으로
`git clone --branch <ref>` 라서 **커밋 SHA 를 넣으면 실패한다**
(`fatal: Remote branch <sha> not found in upstream origin`). 태그나 브랜치만 받는다.
로컬 경로는 그대로 받으므로, SHA 로 체크아웃한 클론을 넘기는 것이 커밋 고정 방법이다.
`Leonxlnx/taste-skill` 은 릴리스 태그가 아예 없어서 이 방법 말고는 고정할 수가 없다.

CLI 버전(`skills@1.5.23`)도 고정한다. `@latest` 는 매번 다른 코드를 받는 것과 같다.
`--copy` 는 심볼릭 링크 대신 실제 파일을 복사한다. git 으로 공유할 때 링크는 저쪽에서 깨진다.

설치 후 **위의 확인 루프를 다시 돌려** `NEED` 가 비었는지 본다. 설치했다는 말은
설치됐다는 증거가 아니다. 남아 있으면 그것만 보고한다.

설치가 끝났으면 "무엇을 깔았다" 한 줄만 남기고 바로 1단계로 넘어간다.
사용자가 설치를 거절했거나 확인 루프에 여전히 남은 게 있으면
**어느 단계가 빠진 채로 진행되는지 밝히고** 1단계로 간다.

## 언어 규칙 — 한국어로 제작한다

**화면에 보이는 모든 문구는 한국어로 쓴다.** 헤드라인, 본문, 버튼 라벨, 네비게이션,
테이블 헤더, 빈 상태 문구, 에러 메시지, 툴팁, alt 텍스트까지 전부.
참조 스타일 가이드가 영문이라는 건 영어로 쓰라는 뜻이 아니다. 토큰만 영문 출처일 뿐이다.
사용자가 명시적으로 영어를 요구할 때만 영어로 간다.

영어로 남기는 것은 이것뿐이다:

- 코드, 커맨드, 파일 경로, 브랜치명, 로그 출력
- 고유명사와 제품명 (Deno, KV, GitHub)
- 업계에서 번역하면 오히려 안 읽히는 약어 (p95, API, CPU, CDN)

**한글은 폰트 스택을 반드시 다시 짠다.** 이걸 빼먹으면 1단계에서 고른 서체가
한글에서 통째로 무너진다. 브리프가 Inter / Space Grotesk 만 지정했더라도:

- 본문·UI: `'Pretendard Variable', Pretendard, 'Inter', ..., 'Apple SD Gothic Neo', 'Malgun Gothic', sans-serif`
- 디스플레이: 영문 디스플레이 서체를 먼저 두되 한글 폴백을 반드시 뒤에 붙인다
- 모노: 한글이 섞이면 `'JetBrains Mono', 'D2Coding', monospace`

**한글 조판 3종은 기본값으로 넣는다.**

- `word-break: keep-all` — 어절 중간에서 끊기는 걸 막는다. 한글 페이지의 최대 티다
- `line-height` 를 영문 기준보다 0.1 올린다. 한글은 같은 값에서 답답해 보인다
- `letter-spacing` 은 음수 트래킹을 한글에 그대로 쓰지 않는다. 디스플레이 사이즈에서 -0.025em 까지만

**금지어는 한국어에도 그대로 적용된다.** design-taste 의 filler verb 금지가
"혁신적인", "차세대", "원활한", "손쉽게", "강력한", "최적화된" 으로 옮겨온 것도 똑같이 금지다.
번역투("~을 제공합니다", "~를 통해")도 피한다. 짧은 서술형으로 쓴다.

**em-dash 금지는 한국어에서 더 엄격하다.** 한국어 문장에 `—` 는 번역투 티다.
쉼표나 마침표로 끊는다.

## 왜 순서를 지키나

네 스킬은 전부 "화면을 잘 만들라"고 하지만 **지시가 서로 충돌한다.**
한꺼번에 켜면 다이얼 값과 토큰 규칙과 감사 체크리스트가 동시에 말을 걸어서
결과가 오히려 뭉개진다.

그래서 섞지 않고 **단계마다 하나씩 주도권을 준다.** 각 스킬이 제일 잘하는 구간이 다르다.

```
방향 ──▶ 다이얼 ──▶ 이미지 ──▶ 구현 ──▶ 감사
 │         │          │         │        │
frontend  design-   image-to-  (코드)  impeccable
-design   taste     code
```

## 1단계 — 방향 (frontend-design)

코드를 한 줄도 쓰기 전에 시각 방향부터 정한다. `frontend-design` 스킬을 읽되
**플랜과 자기반박까지만** 따른다. 이 스킬은 자기 프로세스에 build 단계까지 포함하고 있어서
("start to write the code, following the revised plan exactly") 그대로 따르면 여기서 코딩이 시작되고,
3단계의 이미지 우선 규칙을 위반한 채로 도착한다. 코드는 4단계에서 쓴다.

**읽지 못하면 거기서 사용자에게 알린다.** `frontend-design` 은 기본 탑재가 아니라
설치해야 하는 마켓플레이스 플러그인이고, 0단계 확인 루프는 이걸 잡지 못한다.
없는 채로 감으로 때우고 넘어가면 사용자는 4종이 다 돌았다고 믿는다.
0단계와 같은 규칙이다. 어느 단계가 빠진 채로 진행되는지 밝히고 간다.

정할 것: 색 4~6개, 서체 2종 이상, 레이아웃 골격, 이 화면만의 시그니처 요소 하나.

정하고 나서 **스스로 반박한다.** "이거 AI가 늘 뱉는 그 세 얼굴 중 하나 아닌가?"

```
1. 크림색 배경(#F4F1EA) + 고대비 세리프 + 테라코타 포인트
2. 거의 검은 배경 + 형광 그린/주황 포인트 하나
3. 신문 레이아웃 — 헤어라인 괘선, 라운딩 0, 빽빽한 단
```

셋 중 하나에 걸리면 주제와 무관하게 나온 기본값이라는 뜻이다. 다시 고른다.
**단, 사용자가 그 방향을 직접 지정했으면 그대로 간다.** 원본이 명시한 예외다
("the brief's own words always win, including when it asks for one of these looks").

## 2단계 — 다이얼 (design-taste-frontend)

1단계의 방향을 **수치로 고정**한다. `design-taste-frontend` 스킬을 읽고 그 1장(THE THREE DIALS)을
따른다. 사용자가 쓴 형용사를 다이얼로 번역하는 게 이 단계의 일이다.

| 사용자가 이렇게 말하면 | DESIGN_VARIANCE | MOTION_INTENSITY | VISUAL_DENSITY |
|---|---|---|---|
| "깔끔하게", "심플하게", "에디토리얼" | 5-6 | 3-4 | 2-3 |
| "과감하게", "힙하게", "실험적으로" | 9-10 | 8-10 | 3-4 |
| "프리미엄", "고급스럽게" | 7-8 | 5-7 | 3-4 |
| "신뢰 우선", "공공", "규제 산업" | 3-4 | 2-3 | 4-5 |
| "정보 많이", "데이터 밀도 높게" | (다른 행 따름) | (다른 행 따름) | 8-10 |
| 아무 말 없으면 | 8 | 6 | 4 |

밀도 행은 원본 1.A 에 없다. `8-10` 은 원본 7장의 Cockpit 정의에서 끌어온 이 스킬의 확장이고,
나머지 두 다이얼은 브리프에 맞는 다른 행에서 가져온다. 없는 값을 지어내 채우지 않는다.

기본값 `8 / 6 / 4` 는 `design-taste-frontend` 가 정한 baseline 이다. 임의로 바꾸지 않는다.

**밀도를 거꾸로 잡지 마라.** `VISUAL_DENSITY` 는 높을수록 빽빽하다. `8-10` 이 Cockpit
(빽빽한 패딩, 카드 박스 금지, 1px 선으로 구분, 숫자는 `font-mono`)이고 `1-3` 이 여백 많은 쪽이다.
"정보 많이" 에 낮은 값을 주면 원본의 `VISUAL_DENSITY > 7` 게이트가 영원히 안 걸린다.

**대시보드는 `design-taste-frontend` 가 스코프 밖으로 선언한 영역이다** (원본 머리말:
"Landing pages, portfolios, and redesigns. Not dashboards, not data tables, not multi-step product UI").
대시보드 요청이면 `VISUAL_DENSITY` 만 밀도 행에서 가져오고 `DESIGN_VARIANCE 3-4` /
`MOTION_INTENSITY 2-3` 으로 낮춘다. 보고 화면이지 작품이 아니다.
앞의 두 값은 원본 1.A 의 trust-first 행 그대로다. 원본에 대시보드 행이 아예 없어서
그 행을 대신 쓰는 것이 이 스킬의 판단이다. 디자인 시스템은 원본 2장의 대시보드 행(`@fluentui/*`, `@carbon/react`)을 따른다.

**변수명을 지어내지 않는다.** 다이얼은 이 셋뿐이고, 원본이 문서 전체에서 이 이름 그대로
참조한다. 원본이 예로 든 `LAYOUT_VARIANCE`, `ANIM_LEVEL` 같은 별칭을 만들지 않는다. 없는 다이얼에 값을 잡으면
출력에 아무 영향이 없는데 사용자에게는 잡혔다고 보고된다.

**값은 1-10 정수다.** "스프링", "통제" 같은 말을 값으로 쓰지 않는다. 원본의 게이트가
전부 숫자 비교라서 (`MOTION_INTENSITY > 5` 면 physics, `> 4` 면 실제로 움직여야 하고,
`> 3` 이면 `prefers-reduced-motion` 필수) 문자열을 넣으면 접근성 게이트까지 통째로 빠진다.

정한 값을 **사용자에게 한 줄로 알린다.** 취향은 사람마다 다르고, 숫자로 보여주면
"모션 좀 더" 같은 수정 요청이 쉬워진다.

## 3단계 — 이미지 먼저 (image-to-code)

`image-to-code` 스킬을 읽고 §2 이후의 이미지 생성·분석·추출 규칙을 따른다.
읽지 못하면 어느 단계가 빠진 채로 진행되는지 밝히고 간다.

**§1 ACTIVE BASELINE CONFIGURATION 은 적용하지 않는다.** 이 스킬은 자기 다이얼 9개를
따로 들고 있고 (`ART_DIRECTION`, `SPACING_GENEROSITY`, `VISUAL_DENSITY: 3` 등),
그대로 따르면 2단계에서 정해 사용자에게 보고한 값이 조용히 덮어써진다.
다이얼은 2단계가 이긴다. 여기서만 쓰는 다이얼은 잡지 않는다.

**대시보드·고밀도 경로에서는 §14·§32·§33 도 적용하지 않는다.** `image-to-code` 는 랜딩·마케팅·
포트폴리오 전용이라 다이얼 이름 없이 산문으로 "Do not make the website too dense" (§32) 를 박아두고,
§33 의 기본 섹션 팩도 Hero/Features/Pricing/CTA 같은 랜딩 구조다. 밀도는 2단계 다이얼이 이긴다.

이 단계가 4종 세트의 핵심이자, 혼자 코딩할 때와 제일 크게 갈리는 지점이다.
**바로 코드로 들어가지 않는다.** 참조 이미지를 먼저 만들고, 그걸 뜯어본 다음 옮긴다.

이미지 생성 수단이 있으면:

- 섹션마다 **크고 읽히는 이미지를 따로** 뽑는다. 작은 보드 하나에 몰아넣지 않는다
- 섹션 수에 비해 이미지를 적게 뽑는 게으름을 피한다
- 뽑은 이미지에서 간격·타이포 스케일·정렬·색 관계를 실제로 추출한다

이미지 생성이 안 되는 환경이면 이 단계를 **말로 대신한다.**
섹션별 구성을 글로 먼저 못 박고 넘어간다. 조용히 건너뛰면 3단계가 있으나 마나다.

## 4단계 — 구현

1~3단계에서 정한 것에 맞춰 짓는다. 이 단계에서 새로 디자인 결정을 내리지 않는다.
새 결정이 필요하다는 건 앞 단계가 덜 끝났다는 뜻이므로 거슬러 올라간다.

**2단계 다이얼을 소비하는 게이트는 여기서 적용한다.** `design-taste-frontend` 4~6장을 다시 읽는다.
2단계는 숫자를 정할 뿐이고, 그 숫자를 읽는 규칙은 전부 뒷장에 있다. 여기서 적용하지 않으면
다이얼은 사용자에게 보고만 되고 출력에는 아무 영향이 없다. 최소한 이 다섯은 반드시 건다:

- `MOTION_INTENSITY > 3` — `prefers-reduced-motion` 처리 필수. 원본이 non-negotiable 로 못박은 것
- `MOTION_INTENSITY > 4` — 페이지가 실제로 움직여야 한다. 정적인데 높은 값을 보고하면 거짓말이다
- `MOTION_INTENSITY > 5` — 스프링 물리. 선형 이징 금지
- `VISUAL_DENSITY > 7` — 제네릭 카드 컨테이너 금지. 1px 선으로 구분하고 숫자는 `font-mono`
- `DESIGN_VARIANCE > 4` — 가운데 정렬 히어로 회피

`image-to-code` 가 명시적으로 금지하는 것들이 여기서 자꾸 새어나온다. 짓는 동안 경계한다:

- 카드 안의 카드 안의 카드
- 왼쪽 텍스트 / 오른쪽 이미지 반복
- 거대한 라운딩 컨테이너 도배
- 가짜 인터페이스 용어가 붙은 작은 pill·label·tag
- 첫 화면에 정보 과다. 히어로는 작은 노트북에서도 넓고 읽혀야 한다

## 5단계 — 감사 (impeccable)

`impeccable` 을 읽고 **`audit` → `critique` → 수정 명령** 순으로 돌린다.
읽지 못하면 어느 단계가 빠진 채로 진행되는지 밝히고 간다.

**명령을 명시해야 한다.** 인자 없이 부르면 impeccable 은 메뉴만 띄우고 아무것도 실행하지
않는다 (Routing: `never auto-run a command`). 파이프라인이 마지막 단계에서 멈춘다.

**순서를 바꾸지 마라.** `critique` 는 리포트 뒤에 질문을 붙이고 거기서 턴을 끝내는 명령이라
(`The question is the LAST thing in the response`) 뒤에 뭘 이어 붙이면 그 산출물이 통째로
묻힌다. `audit` 이 먼저다. 커버 범위도 다르다 — `audit` 은 a11y·성능·반응형,
시각 계층과 빈 상태는 `critique` 쪽이라 둘 다 필요하다.

**둘 다 고치는 명령이 아니다.** `audit` 은 명시적으로 `Don't fix issues; document them for
other commands to address` 다. 수정은 `polish` 나 audit 리포트가 추천한 명령으로 따로 돌린다.
`reference/craft-floor.md` 는 그 **수정 명령 직전**에 읽는다. audit·critique 단계에서 읽지 않는다
(원본: `Do not load it for planning-only work`).

세션에서 처음이면 `node <skill-base-dir>/scripts/context.mjs` 를 한 번 돌린다.
`<skill-base-dir>` 은 런타임이 이 스킬에 대해 보고하는 경로이고, 없으면
`.claude/skills/impeccable/scripts` 가 폴백이다. cwd 는 프로젝트 루트에 둔다.

자기 눈으로 훑고 "impeccable 감사 완료" 라고 보고하는 것이 이 단계에서 제일 흔한 거짓말이다.
찾은 것 중 **실제로 고친 것과 남긴 것을 구분해서** 보고한다.
전부 고쳤다는 보고는 대개 안 본 것이다.

## 끝나고 남기는 말

짧게. 무엇을 만들었는지, 2단계 다이얼 값이 뭐였는지, 감사에서 뭘 고쳤는지.
다이얼 값을 남기는 이유는 다음 요청이 "모션 좀 줄여줘" 한 마디로 끝나게 하려는 것이다.
