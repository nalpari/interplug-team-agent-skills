---
name: ip-design-skill
description: '디자인 스킬 4종(frontend-design, design-taste-frontend, image-to-code, impeccable)을 방향→다이얼→이미지→구현→감사 순서로 물려서 한국어 화면을 설계·구현·감사한다. "디자인해줘", "화면 만들어줘", "UI 만들어줘", "랜딩페이지 만들어줘", "대시보드 화면 짜줘", "design this page", "build this UI" 처럼 화면·UI 제작을 요청할 때 사용한다. 빠진 스킬은 설치 명령만 안내하고 직접 설치하지 않는다. Claude Code 전용.'
---

# 디자인 4종 세트로 만든다

사용자가 매번
*"design-taste-frontend, image-to-code, impeccable, frontend-design 적극 활용해서 디자인해줘"*
라고 길게 치는 걸 없애는 것이 이 스킬의 목적이다. 그러니 **묻지 말고 바로 시작한다.**
무엇을 만들지 알 수 없을 때만 한 줄로 물어본다.

## 0단계: 준비 — 확인만 한다

**4종을 자동으로 설치하지 않는다.** 확인해서 빠진 게 있으면 설치 명령을 보여주고,
실행은 사용자에게 맡긴다. 서드파티 SKILL.md 는 에이전트가 그대로 따르는 지시문이고
`impeccable` 은 `.mjs` 실행 스크립트와 훅 런타임까지 같이 깔린다.
"화면 만들어줘" 한 마디가 그 설치의 승인이 될 수는 없다. 머신당 한 번 있는 일을
자동화하려고 에이전트에게 파괴적인 설치 권한을 주지 않는다.

확인 경로는 **Claude Code 기준(`.claude/skills/`)이다.** Codex·Cursor 에서 실행 중이면
이 루프를 건너뛰고 4종이 설치돼 있는지 사용자에게 물어본 뒤 1단계로 간다.

`frontend-design` 은 스킬이 아니라 마켓플레이스 플러그인이라 `~/.claude/plugins/` 밑에 깔린다.
여기서는 안 잡히므로 1단계에서 직접 확인한다.

```bash
for s in design-taste-frontend image-to-code impeccable; do
  [ -f ".claude/skills/$s/SKILL.md" ] || [ -f "$HOME/.claude/skills/$s/SKILL.md" ] \
    || echo "MISSING:$s"
done
```

아무것도 안 나오면 **이 단계를 통째로 건너뛰고 1단계로 간다.** 보고도 하지 않는다.
준비됐다는 말은 사용자가 원한 결과물이 아니다.

빠진 게 있으면 해당 줄만 사용자에게 보여주고 **직접 실행하라고 안내한다.**

```bash
# design-taste-frontend, image-to-code
d=$(mktemp -d) && git clone -q --filter=blob:none https://github.com/Leonxlnx/taste-skill "$d" \
  && git -C "$d" checkout -q ccbc15639c97057cbfcf32ecebc38ef716e4bb37 \
  && npx --yes skills@1.5.23 add "$d" --skill design-taste-frontend --agent claude-code --yes --copy \
  && npx --yes skills@1.5.23 add "$d" --skill image-to-code --agent claude-code --yes --copy; rm -rf "$d"

# impeccable
d=$(mktemp -d) && git clone -q --filter=blob:none https://github.com/pbakaus/impeccable "$d" \
  && git -C "$d" checkout -q 63b04e2530f5c7b41ea83c133daab24f34912456 \
  && npx --yes skills@1.5.23 add "$d" --skill impeccable --agent claude-code --yes --copy; rm -rf "$d"
```

명령과 함께 알려줄 것:

- **`--copy` 는 목적지 디렉토리를 지우고 다시 만든다.** `.claude/skills/` 에 같은 이름으로
  뭘 만들어 뒀으면 먼저 옮긴다. 그 안의 파일은 복구할 수 없다
- 설치하면 `skills-lock.json` 에 방금 지운 temp 경로가 적힌다. 커밋하지 말고 되돌린다
- 커밋 SHA 로 고정한 이유는 `skills add <repo>@<ref>` 가 내부적으로 `git clone --branch <ref>`
  라서 SHA 를 못 받기 때문이다. SHA 로 체크아웃한 클론을 로컬 경로로 넘기는 것이 고정 방법이다

**사용자가 설치하지 않고 그냥 진행하라고 하면 그렇게 한다.** 어느 단계가 빠진 채로
가는지만 밝히고 1단계로 넘어간다.

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
