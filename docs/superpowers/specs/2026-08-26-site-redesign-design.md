# parameterfreak.com 전체 리디자인 — 설계

날짜: 2026-08-26
방향: **Signal + 블루** (시안 v3, https://claude.ai/code/artifact/6c1df37d-57de-4339-8272-4a9311325439)

## 목표

- 사이트 전체를 하나의 시각 언어로 통일한다: 흰 바탕, 큰 검정 활자, 여백, 포인트 색은 제품(H-MAS v0.7 UI)의 전기 블루 하나.
- 홈은 "여러 가지를 하는 팀"(서빙 플랫폼 / 임베딩·이상탐지 연구 / AI 소프트웨어)을 같은 무게로 보여준다.
- 제품 스크린샷(v0.7 라이트)이 사이트의 일부처럼 보이게 한다.
- academicpages 테마를 걷어내지 않고, 토큰과 크롬(마스트헤드·푸터)·글 상세 레이아웃을 교체하는 것으로 72개 글 페이지를 한 번에 바꾼다.

## 범위

| 대상 | 파일 | 작업 |
|---|---|---|
| 디자인 토큰 | `_sass/_themes.scss`, `_sass/theme/_default_light.scss`, `_sass/_landing.scss` | 색·활자 변수 교체 |
| 폰트 | `_includes/head/custom.html` | Google Fonts 링크 추가 |
| 마스트헤드 | `_includes/masthead.html`, `_sass/layout/_masthead.scss`, `_sass/layout/_navigation.scss` | 마크업·스타일 교체 (로고 잘림 버그 해결) |
| 푸터 | `_includes/footer.html`, `_includes/footer/custom.html`, `_sass/layout/_footer.scss` | 마크업·스타일 교체 |
| 글 상세 | `_layouts/single.html`, `_sass/layout/_page.scss` | 레이아웃 교체 (좌측 목차 + 본문 680px) |
| 홈 | `_pages/index.html` | 시안 v3에 맞춰 재작성 |
| H-MAS 랜딩 | `_pages/hmas.html` | 라이트 톤으로 재스타일, 스크린샷 v0.7로 교체 |
| 목록 3개 | `_pages/portfolio.html`, `_pages/changelog.html`, `_pages/year-archive.html` | 토큰 따라 자동 반영, 마크업 소폭 정리 |
| 이미지 | `images/portfolio/hmas/v0.7/*.png`, `_config.yml` `og_image` | v0.7 스크린샷 사용 |

범위 밖 (그대로 둠): `_publications`/`_talks`/`_teaching` 샘플, `/cv/`, `/markdown/`, `/terms/`, `/sitemap/` 등 academicpages 잔재. 새 글 상세 레이아웃이 자동 적용되며, 별도 손대지 않는다. 다크 모드는 지원하지 않는다 (기존 force-light 스크립트 유지).

## 디자인 토큰

```scss
// 색
--pf-bg:    #ffffff;   // 바탕
--pf-ink:   #16181d;   // 본문·제목
--pf-grey:  #4b4f57;   // 보조 텍스트
--pf-mute:  #6b6f78;   // 메타·캡션
--pf-line:  #e9eaec;   // 구분선
--pf-soft:  #f7f7f8;   // 옅은 섹션 배경 (목록 페이지 카드 영역 등)
--pf-blue:  #063ad7;   // 포인트 (링크, 버튼, 강조 단어, 상태 점)
--pf-blue-soft: #e8edfb; // 블루 배경 (태그, 배지)

// 활자
$display: "Gothic A1", "Noto Sans KR", sans-serif;      // 제목. 900/700, letter-spacing -0.03em
$body:    "Noto Sans KR", -apple-system, sans-serif;     // 본문. 400/500/700
$mono:    "IBM Plex Mono", Menlo, monospace;             // 날짜·코드
```

타입 스케일 (데스크톱 / 모바일):
- h1 홈·랜딩 히어로 54px / 34px, h1 글 제목 38px / 28px
- h2 섹션 32px / 26px, h2 본문 22px
- 본문 16px, 행간 1.75; 글 본문 15.5px, 행간 1.85
- 라벨 12px, letter-spacing .14em, 대문자, 블루

academicpages의 `--global-*` 변수는 위 토큰을 가리키도록 재정의한다 (`--global-text-color: var(--pf-ink)`, `--global-link-color: var(--pf-blue)` 등). `$global-font-family`/`$header-font-family`도 위 활자로 바꾼다. 이렇게 하면 손대지 않은 테마 컴포넌트(잔재 페이지, 알림 박스, 표)도 새 팔레트로 렌더링된다.

Google Fonts: `Gothic A1` 700·900, `Noto Sans KR` 400·500·700, `IBM Plex Mono` 400·500. `display=swap`.

## 컴포넌트

### 마스트헤드
- 좌: `parameterfreak` 워드마크 (Gothic A1 900, 16px, 소문자). 우: Products / Changelog / Blog + 블루 알약 버튼 `문의하기` (`mailto:contact@parameterfreak.com`).
- 높이 64px, 흰 배경, 하단 1px `--pf-line`. sticky 아님.
- greedy-nav JS와 햄버거 제거. 항목이 4개뿐이라 모바일에서도 한 줄(폰트 13px, 간격 축소)로 수용한다. 600px 이하에서 `문의하기` 버튼은 숨긴다.
- 현재 페이지 링크는 잉크색, 나머지는 `--pf-grey`.
- 로고 잘림 버그의 원인(`.masthead__inner-wrap` 좌측 패딩/`greedy-nav` 오버플로)은 마크업 교체로 사라진다.

### 푸터
- 1행: 워드마크 + 한 줄 설명("AI 시스템을 설계하고 만듭니다") / 링크 열: Products, Changelog, Blog, Feed, Sitemap / 연락: contact@parameterfreak.com
- 2행: `© 2026 parameterfreak` (연도는 `site.time`). Jekyll/AcademicPages 크레딧 문구는 제거. MathJax 스크립트는 `footer/custom.html`에 유지.

### 글 상세 (`single.html`)
`_posts`, `_changelog`, `_portfolio` 및 잔재 페이지 전부에 적용.

구조:
```
<div class="art">
  <aside class="art__toc">    ← ≥1024px에서만 표시, sticky
    ON THIS PAGE + h2 목록
  </aside>
  <article class="art__body"> ← max-width 680px
    crumb   : 컬렉션 라벨 (Changelog · H-MAS / Blog / Products)
    h1      : 제목
    meta    : 날짜(mono) · 읽는 시간
    content
    tags    : 알약 태그 (page.tags)
    prev/next : 같은 컬렉션 내 이전/다음 글
  </article>
</div>
```
- 목차는 본문 `h2`에서 생성. 레이아웃은 마크다운 처리 대상이 아니라 kramdown `{:toc}`를 못 쓰므로, 10줄 내외의 인라인 JS로 `h2[id]`를 읽어 목록을 만든다. h2가 2개 미만이면 목차를 숨긴다.
- crumb 라벨 규칙: `page.collection == "changelog"` → `Changelog · {{ categories[0] }}`, `posts` → `Blog`, `portfolio` → `Products`, 그 외 → 없음.
- 제거: 사이드바 author-profile 호출, breadcrumbs, 소셜 공유 버튼, 댓글 블록, related posts, 인용/논문 링크 블록. `_config.yml` defaults의 `share`/`comments`/`related`는 더 이상 참조되지 않으므로 함께 지운다.
- 본문 스타일: 링크 블루 밑줄, `code` 회색 배경, `pre` 잉크색 배경 + 흰 글자(제품 Pod 로그 톤), 표는 `overflow-x:auto` 래퍼, 이미지 `max-width:100%` + 둥근 모서리 8px + 1px 선.

### 랜딩 컴포넌트 (`_landing.scss`)
기존 `.lp-*` 클래스명을 유지하고 값만 바꾼다. 주요 변경:
- `.lp-hero`: 그라데이션 제거, 흰 배경, 잉크색 글자. `.lp-hero__title`에 `<u>` 강조 단어는 블루.
- `.lp-card`: 배경·테두리·그림자 없음. 위쪽 2px 잉크색 선 + 위 패딩 20px. `.lp-card__icon`(이모지) 제거.
- `.lp-card--product` (Products 목록): 흰 카드 + 1px 선 + 스크린샷 썸네일 유지.
- `.lp-btn--primary`: 블루 알약. `.lp-btn--ghost`/`--line`: 1px `#cfd1d6` 알약.
- `.lp-eyebrow`: 블루, 12px, 대문자.
- `.lp-post` (목록 행): 좌 제목·요약, 우 날짜(mono `--pf-mute`). 행 사이 1px 선, 첫 행 위 잉크색 1px 선.
- `.lp-stat__num`: 블루, Gothic A1 900, 40px.
- `.lp-section--dark`, `.lp-final`의 어두운 배경 제거 → 흰 배경 또는 `--pf-soft`.
- `.lp-status`(신규): 블루 점 + 13px 텍스트. 홈 히어로 상단 한 줄용.

## 페이지

### 홈 `/`
1. 히어로 (2열, 820px 이하 1열): 좌 — `.lp-status` "H-MAS v0.7 Feature Preview 출시" → h1 "AI 시스템을 / 설계하고, <u>만듭니다.</u>" → 리드 → 버튼 `H-MAS 알아보기`(→ /hmas/), `제품 전체`(→ /solution/). 우 — `02-dashboard.png` 스크린샷 (흰 프레임, 그림자).
2. What we do: 카드 3개 — AI 서빙 플랫폼(→/hmas/), 임베딩·이상탐지 연구(→/posts/), AI 소프트웨어(→/solution/). 문구는 현재 `index.html` 유지.
3. Changelog: 제품 무관 최신 릴리스 4행 (`site.changelog | sort date | reverse | limit 4`), 우측에 제품명 블루. 하단 `전체 Changelog →`.
4. Blog: 최신 글 3행. 하단 `블로그 →`.
5. Contact: h2 "함께 이야기해요" + 이메일 버튼. 현재 유지.

현재 홈의 FLAGSHIP 섹션(썸네일 카드 + H-MAS 릴리스 3개)은 제거한다. 히어로 스크린샷이 그 역할을 하고, Changelog 섹션이 릴리스를 보여준다.

### H-MAS 랜딩 `/hmas/`
- 히어로: 중앙 정렬. 라벨 "H-MAS · Hardware-aware Multi-Cluster AI Serving Platform" → h1(현재 문구, `<u>최적의 하드웨어에</u>` 강조) → 리드 → 버튼 2개 → 알약 배지 3개(v0.7 Feature Preview, 저작권 등록, 토폴로지 인식 스케줄링) → `02-dashboard.png` 큰 스크린샷(max-width 960px).
- 이하 8개 섹션은 내용·순서 유지, 스타일만 토큰을 따른다. 숫자 카드는 검정 상단선 규칙.
- 스크린샷 교체: `pitch/dashboard.png` → `v0.7/02-dashboard.png`, `pitch/deploy-form.png` → `v0.7/05-deploy.png`, `pitch/monitoring.png` → `v0.7/08-monitoring.png`. `architecture.png`, `copyright-2604.png`는 유지. "웹 콘솔에서 모델만 고르면 끝" 섹션의 `.lp-shots` 그리드는 `04-models.png`(모델 저장소) → `05-deploy.png`(배포 마법사) → `08-monitoring.png`(모니터링) 3장으로 구성한다.
- 라이트/다크 지원을 보여주는 곳: 기존 "바로 도입할 수 있는 검증된 제품" 섹션에 `02-dashboard.png`와 `02-dashboard-dark.png`를 나란히 한 쌍 배치. 다크 스크린샷 사용처는 이곳뿐이다.

### Products `/solution/`
- 히어로 문구 유지, 카드 2개(H-MAS → /hmas/, PaperOps → /solution/paperops/). H-MAS 썸네일을 `v0.7/02-dashboard.png`로 교체 (`_portfolio/H-MAS.md`의 `header.teaser`).

### Changelog `/changelog/`, Blog `/posts/`
- 마크업 유지. `.lp-post` 스타일 변경으로 자동 반영. 연도/제품 헤더(`.lp-year`)는 Gothic A1 700, 20px, 위에 잉크색 1px 선.

### 그 외
- `_config.yml` `og_image` → `portfolio/hmas/v0.7/02-dashboard.png`.
- `_portfolio/H-MAS.md`(549줄 문서)와 `paperops.md`는 새 글 상세 레이아웃으로 렌더링. 본문 안의 `pitch/*.png` 참조는 v0.7로 교체.

## 반응형
- 브레이크포인트: 1024px(목차 표시), 820px(히어로 1열, 카드 3→1열), 600px(마스트헤드 축소, `문의하기` 숨김).
- 스크린샷은 `max-width:100%`, 히어로 1열에서는 텍스트 아래로 내려간다.

## 검증
- 로컬 빌드 환경 구축: Homebrew Ruby(`brew install ruby`) + `bundle install` + `bundle exec jekyll serve`. 이 환경이 없으면 페이지를 눈으로 확인할 수 없으므로 첫 작업으로 둔다.
- 각 단계마다 `bundle exec jekyll build`가 오류 없이 끝나고, Playwright로 `/`, `/hmas/`, `/solution/`, `/changelog/`, `/posts/`, 글 상세 1개(`/changelog/h-mas/2026-08-02/`), 잔재 페이지 1개(`/cv/`)를 데스크톱(1280px)·모바일(390px) 스크린샷으로 확인한다.
- 작업은 `redesign` 브랜치에서 하고, 마지막에 master로 머지·푸시 후 라이브에서 같은 URL을 다시 확인한다. 기존 URL·리다이렉트는 이번 작업에서 바뀌지 않는다.

## 작업 순서 (구현 계획의 뼈대)
1. 로컬 빌드 환경
2. 토큰·폰트 (`_themes.scss`, `_default_light.scss`, `head/custom.html`) — 이 단계만으로 전 페이지 색·활자가 바뀐다
3. 마스트헤드·푸터
4. 글 상세 레이아웃 (`single.html` + `_page.scss`)
5. `_landing.scss` 재작성
6. 홈
7. H-MAS 랜딩 + 스크린샷 교체
8. Products/Changelog/Blog 목록 점검, `og_image`, portfolio 문서 이미지 교체
9. 전체 스크린샷 검증 → 머지·푸시 → 라이브 확인
