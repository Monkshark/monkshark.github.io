---
title: "Hugo + Stack 테마로 블로그 만들기"
description: "Hugo 설치부터 Stack 테마 커스터마이징, 스크롤 애니메이션, GitHub Pages 배포까지"
date: 2026-04-04
slug: hugo-stack-blog-setup
categories:
    - TIL
tags:
    - Hugo
    - Stack
    - GitHub Pages
    - CSS
---

## Hugo 프로젝트 생성

```bash
hugo new site monkshark.github.io --format yaml
```

Hugo는 기본적으로 `config.toml`을 생성하는데, `--format yaml`을 붙이면 `hugo.yaml`로 시작한다. YAML이 TOML보다 들여쓰기가 직관적이라 선호한다.

```bash
git init
git submodule add https://github.com/CaiJimmy/hugo-theme-stack themes/hugo-theme-stack
```

Stack 테마를 Git submodule로 추가했다. 테마를 직접 복사하는 대신 submodule로 관리하면, 테마 업데이트를 `git submodule update`로 간단히 반영할 수 있다.

## hugo.yaml 설정

Stack 테마는 설정이 많은데, 핵심만 정리하면:

```yaml
baseURL: https://monkshark.github.io/
languageCode: ko
title: monkshark.dev
theme: hugo-theme-stack

params:
  sidebar:
    subtitle: "풀스택 개발자가 되고싶은 사람"

  article:
    toc: true           # 목차 자동 생성
    readingTime: true   # 읽는 시간 표시

  colorScheme:
    toggle: true        # 다크/라이트 전환 버튼
    default: auto       # 시스템 설정 따라감

  widgets:
    homepage:
      - type: search
      - type: archives
      - type: categories
      - type: tag-cloud
```

`colorScheme.default: auto`로 설정하면 사용자의 시스템 다크 모드 설정을 따라가되, 토글 버튼으로 수동 전환도 가능하다.

permalink를 `/p/:slug/`로 설정해서 URL이 깔끔하게 나오도록 했다. 날짜가 URL에 들어가면 길어지고, 나중에 글 날짜를 수정하면 URL이 바뀌어서 링크가 깨질 수 있다.

## 커스텀 스크롤 애니메이션

Stack 테마는 깔끔하지만 정적이다. 스크롤할 때 요소들이 자연스럽게 등장하는 애니메이션을 추가하고 싶었다.

### CSS — `assets/scss/custom.scss`

핵심 아이디어: `body.anim-ready` 클래스가 있을 때만 요소를 숨기고, JS가 스크롤 위치에 따라 `.is-visible`을 추가하면 등장한다.

```scss
// JS가 anim-ready를 추가한 후에만 숨김 처리
// → JS가 실패해도 콘텐츠는 보인다 (progressive enhancement)
body.anim-ready .article-list article {
    opacity: 0;
    transform: translateY(30px) scale(0.98);
    transition: opacity 0.6s cubic-bezier(0.16, 1, 0.3, 1),
                transform 0.6s cubic-bezier(0.16, 1, 0.3, 1);
}

body.anim-ready .article-list article.is-visible {
    opacity: 1;
    transform: translateY(0) scale(1);
}
```

`body.anim-ready`를 가드 조건으로 쓰는 이유가 있다. 처음에는 CSS에서 바로 `opacity: 0`을 적용했는데, JS 로딩이 지연되면 **페이지 전체가 빈 화면**으로 보이는 문제가 있었다. JS가 성공적으로 로드된 후에만 `body`에 `anim-ready` 클래스를 추가하고, 그때부터 애니메이션이 작동하게 했다. JS가 실패하면? 클래스가 안 붙으니까 모든 요소가 원래대로 보인다.

카드 호버, 사이드바 위젯 등장, 헤딩 슬라이드, 코드 블록 fade-in, 네비게이션 밑줄 애니메이션도 같은 패턴으로 추가했다. 전부 순수 CSS + `cubic-bezier(0.16, 1, 0.3, 1)` (ease-out-expo와 유사한 커브)로, 외부 라이브러리 없이 구현했다.

그리고 `prefers-reduced-motion` 미디어 쿼리로 모션 감소 설정을 존중한다:

```scss
@media (prefers-reduced-motion: reduce) {
    body.anim-ready .article-list article,
    body.anim-ready .animate-on-scroll,
    /* ... 기타 요소들 */  {
        opacity: 1;
        transform: none;
    }
}
```

### JS — `layouts/partials/footer/custom.html`

Intersection Observer API로 요소가 뷰포트에 들어오는 시점을 감지한다:

```javascript
document.body.classList.add('anim-ready');

var observer = new IntersectionObserver(function(entries) {
    entries.forEach(function(entry) {
        if (entry.isIntersecting) {
            entry.target.classList.add('is-visible');
            observer.unobserve(entry.target);  // 한 번만 실행
        }
    });
}, { threshold: 0.05, rootMargin: '0px 0px -30px 0px' });
```

카드 목록에는 `transitionDelay`를 인덱스 × 0.1초로 설정해서 순차적으로 등장하는 스태거(stagger) 효과를 줬다. 한꺼번에 나타나면 밋밋하고, 하나씩 올라오면 시선이 자연스럽게 따라간다.

## GitHub Actions 배포

```yaml
# .github/workflows/hugo.yml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: "0.160.1"
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb \
            https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Build with Hugo
        run: hugo --gc --minify
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
```

`master` 브랜치에 push하면 Hugo가 빌드하고, GitHub Pages에 자동 배포된다. `submodules: recursive`가 중요한데, Stack 테마가 submodule로 들어가 있어서 이걸 빠뜨리면 빌드 시 테마를 찾지 못한다.

Hugo Extended 버전을 써야 SCSS 컴파일이 된다. 일반 Hugo로 빌드하면 `custom.scss`가 무시되어 커스텀 스타일이 적용되지 않는다.

## 카테고리 아카이브 커스터마이징

카테고리별 아카이브 페이지(`/archives/`)에서 타일 카드에 카테고리 아이콘을 배경처럼 깔아줬다. 아이콘이 왼쪽과 하단으로 살짝 잘리면서 카드 뒤에 은은하게 보이는 효과.

```scss
.article-list--tile article.has-image .article-image {
    position: absolute;
    left: -40px;
    top: -15px;
    z-index: 0;

    img {
        width: 240px !important;
        height: 240px !important;
        border-radius: 24px;
        opacity: 0.45;
    }
}
```

`overflow: hidden`으로 카드 영역을 벗어나는 부분을 자르고, `z-index`로 텍스트가 아이콘 위에 올라오게 했다. 다크 모드에서는 `opacity: 0.3`으로 더 어둡게 처리한다.

## 정리

| 항목 | 선택 |
|------|------|
| 정적 사이트 생성기 | Hugo v0.160.1 Extended |
| 테마 | Stack (git submodule) |
| 배포 | GitHub Pages + GitHub Actions |
| 커스텀 애니메이션 | CSS + Intersection Observer (외부 라이브러리 없음) |
| 비용 | $0 |
