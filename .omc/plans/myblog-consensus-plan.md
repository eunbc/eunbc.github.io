# Consensus Plan — Hugo PaperMod 개인 포트폴리오 블로그

> Source spec: `.omc/specs/deep-interview-myblog.md` (ambiguity 10.5%, PASSED)
> Consensus iterations: 1 (Architect + Critic both APPROVED_WITH_IMPROVEMENTS, all 4 major findings absorbed)
> Mode: SHORT RALPLAN-DR (non-deliberate; risk profile is low)

## RALPLAN-DR Summary

### Principles
1. **테마 신뢰**: PaperMod 검증된 디자인을 기반으로, 색상·폰트만 손본다. 풀커스텀 금지.
2. **콘텐츠 큐레이션 = 포트폴리오**: 양보다 질. 자랑스러운 글 5–10개로 시작.
3. **0원 운영**: 도메인·호스팅·유지비 모두 무료. GitHub Pages + 서브도메인.
4. **자동 빌드**: GitHub Actions로 push → 자동 배포.
5. **점진적 확장**: 1차에서는 댓글·SEO·커스텀 도메인 제외. 필요 시 v2.
6. **커스터마이징 천장 (NEW, 원칙 1·5 긴장 해소)**:
   - `assets/css/extended/custom.css` ≤ 50줄
   - `layouts/partials/extend_head.html`, `extend_footer.html` 외 다른 partial 오버라이드 금지
   - `layouts/_default/*` 오버라이드 금지 (이걸 넘으면 v2)

### Decision Drivers (Top 3)
1. **시간 예산 = 주말** → 가장 빠른 배포 경로
2. **이직용 first impression** → 30초 안에 "정리력 있음"이 보여야 함
3. **유지비/관리 부담 0** → 빌드 자동화, 의존성 최소화

### Viable Options Considered
- **Option A (선택)**: Hugo + PaperMod + GitHub Pages + GitHub Actions
- **Option B**: Astro (AstroPaper) + Cloudflare Pages — Architect 스틸맨에서 강력한 후보로 떠올랐음. npm semver 기반 테마 라이프사이클, MDX 마이그레이션 친화성, preview deploy. 하지만 사용자가 명시적으로 GitHub Pages를 선택했고 Go 친숙도를 선호함 → **invalidated by user constraint**, 단 Architect의 우려사항(테마 라이프사이클·마이그레이션 도구)은 본 계획의 v8.x 태그 핀 + pandoc 도입으로 흡수.
- **Option C**: Hugo + Netlify free tier — 1차 검토에서 누락된 옵션. Cloudflare/Vercel과 동급 DX, 무료 티어. 하지만 별도 계정 + GitHub Pages 명시 선택 → **invalidated**.
- **Option D**: Hugo `hugo deploy` + AWS S3 — "0원" 위반 → **invalidated**.

선택 근거: 5개 원칙(특히 0원 운영·자동 빌드)을 만족하면서 사용자 명시 선택과 일치하는 유일한 옵션.

---

## Requirements Summary
- 빈 디렉토리 `/Users/eunbi/GolandProjects/myblog`에 Hugo 정적 사이트 부트스트랩
- PaperMod 테마 적용 (Git submodule, **태그 핀**)
- 색상/폰트 1회 이상 커스터마이징
- About 페이지에 본인 소개·연락처·GitHub 링크
- Confluence에서 옮긴 마크다운 글 1개 이상 게시
- GitHub Actions 워크플로로 push → 자동 배포
- 외부 접근 가능한 URL에서 사이트 노출
- 다크모드/태그/검색 (PaperMod 기본 제공)

## Acceptance Criteria

각 항목을 `[automated]` (명령으로 검증) 또는 `[user-confirm]` (시각/수동 확인)으로 분류.

- [ ] **[automated]** `hugo version`이 v0.120 이상 출력
- [ ] **[automated]** `git remote -v`에 `<repo-url>` 등록됨
- [ ] **[automated]** `themes/PaperMod/` submodule 존재 + **명시적 태그에 핀**됨 (`git -C themes/PaperMod describe --tags --exact-match` 성공)
- [ ] **[automated]** `hugo --gc --minify` 명령이 에러 없이 `public/` 생성
- [ ] **[automated]** `public/index.html`이 PaperMod 마크업 포함 (`grep -q 'class="entry"' public/index.html` 또는 PaperMod 식별 마커)
- [ ] **[automated]** `hugo.toml`에 `theme = "PaperMod"` 명시 (baseURL은 deployment 모드에 따라 분기 — 아래 Phase A 참조)
- [ ] **[automated]** `content/posts/` 하위에 마크다운 글 1개 이상
- [ ] **[automated]** About 페이지(`content/about.md` 또는 `content/about/_index.md`) 존재 + GitHub 링크 텍스트 포함 (`grep -q "github.com" content/about*`)
- [ ] **[automated]** `assets/css/extended/custom.css`에 색상/폰트 오버라이드 1개 이상 (`grep -E "(--primary|font-family)" assets/css/extended/custom.css`)
- [ ] **[automated]** `assets/css/extended/custom.css` 줄수 ≤ 50 (`wc -l < assets/css/extended/custom.css`)
- [ ] **[automated]** `.github/workflows/hugo.yml` 존재 + `submodules: recursive` 플래그 사용
- [ ] **[user-confirm]** GitHub repo Settings → Pages → Source = `GitHub Actions`로 설정됨
- [ ] **[user-confirm]** push 후 Actions 워크플로 성공 (녹색 체크)
- [ ] **[user-confirm]** 배포 URL에서 메인 + 글 1개 진입 가능
- [ ] **[user-confirm]** 모바일 너비(≤375px)에서 가로 스크롤 없음
- [ ] **[user-confirm]** 30초 first-impression 테스트 통과 (이름 + 직무 + 글 목록이 한 화면) — **non-blocking**

## Implementation Steps

### Phase A — 사용자 입력 수집 (Autopilot Phase 2 시작 시)

**`AskUserQuestion`로 한 번에 묻기 (최대 4개 질문):**

1. **GitHub username** (프리텍스트)
2. **Deployment 모드** (단일 선택):
   - **User site**: repo 이름 = `<username>.github.io`, 배포 URL = `https://<username>.github.io/`, baseURL path = 루트
   - **Project site**: repo 이름 = `myblog` (또는 원하는 이름), 배포 URL = `https://<username>.github.io/myblog/`, baseURL path = `/myblog/`
   - **권장**: User site (1인 1개만 가능하지만 URL이 깔끔하고 포트폴리오 first impression에 유리)
3. **메인 색상** (프리텍스트, hex 또는 톤 단어 — 예: `#2d3a4f`, "warm beige")
4. **폰트 전략** (단일 선택):
   - System (가장 가벼움, 한글: `-apple-system`/`Apple SD Gothic Neo` fallback)
   - Pretendard via CDN (한국어 가독성 ↑, 한 줄 추가)
   - Google Fonts (영문만, 한 줄 추가)

추가로 (선택, 첫 게시글):
- 첫 Confluence 글 1개의 마크다운 텍스트 또는 "스켈레톤으로 placeholder 글 1개 생성" 옵션

### Phase B — 로컬 부트스트랩

5. **사전 점검**: `hugo version`. 없거나 v0.120 미만이면 `brew install hugo` 안내 후 사용자 위임 (체크포인트).
6. **`.omc/` 보호 확인**: Hugo는 자신이 생성하는 디렉토리만 건드리지만, 안전을 위해 `hugo new site . --force` 실행 전 `.omc/` 백업 제안 (실제로는 Hugo가 건드리지 않음 — 1줄 코멘트만 남김).
7. **Hugo 사이트 부트스트랩**: `hugo new site . --force`
8. **Git init + .gitignore**:
   - `git init -b main`
   - `.gitignore`: `public/`, `resources/`, `.hugo_build.lock`
9. **PaperMod submodule 추가 + 태그 핀**:
   ```bash
   git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
   cd themes/PaperMod
   # 최신 안정 태그 확인 → v8.x 계열
   git fetch --tags
   LATEST_TAG=$(git tag -l 'v*' | sort -V | tail -1)
   git checkout "$LATEST_TAG"
   cd ../..
   git add themes/PaperMod
   ```
   README에 핀된 태그 명시.

10. **`hugo.toml` 작성** (deployment 모드 분기):
    - User site:
      ```toml
      baseURL = "https://<username>.github.io/"
      ```
    - Project site:
      ```toml
      baseURL = "https://<username>.github.io/<repo>/"
      ```
    - 공통:
      ```toml
      languageCode = "ko-kr"
      defaultContentLanguage = "ko"
      title = "<사이트 제목>"
      theme = ["PaperMod"]
      enableRobotsTXT = true

      [params]
      defaultTheme = "auto"
      ShowReadingTime = true
      ShowShareButtons = false
      ShowToc = true
      ShowCodeCopyButtons = true

      [outputs]
      home = ["HTML", "RSS", "JSON"]

      [[menu.main]]
      identifier = "posts"
      name = "Posts"
      url = "/posts/"
      weight = 10

      [[menu.main]]
      identifier = "about"
      name = "About"
      url = "/about/"
      weight = 20

      [[menu.main]]
      identifier = "tags"
      name = "Tags"
      url = "/tags/"
      weight = 30
      ```

### Phase C — 콘텐츠 스캐폴드

11. **About 페이지**: `content/about.md`
    ```yaml
    ---
    title: "About"
    layout: "page"
    url: "/about/"
    ---
    ```
    본문: 이름, 한 줄 소개, GitHub 링크, 이메일.

12. **첫 게시글**: `content/posts/<slug>.md` (사용자 제공 텍스트 또는 placeholder)
    front matter 템플릿:
    ```yaml
    ---
    title: "<글 제목>"
    date: 2026-05-09
    draft: false
    tags: ["tag1"]
    summary: "한 줄 요약"
    slug: "english-kebab-case"
    ---
    ```

13. **`README.md`**: 로컬 실행, 글 추가, 빌드/배포, **PaperMod 태그 업그레이드 의식**, 마이그레이션 가이드 링크.

### Phase D — 디자인 튜닝 (커스터마이징 천장 ≤ 50줄)

14. **`assets/css/extended/custom.css`**:
    ```css
    :root {
      --primary: <user-hex>;
      --secondary: <derived>;
      /* PaperMod의 light/dark theme variable에 맞춰 오버라이드 */
    }
    body {
      font-family: <user-font-stack>, -apple-system, BlinkMacSystemFont, sans-serif;
    }
    ```
    줄수 ≤ 50 강제 (CI에서 검증 가능, 로컬도 `wc -l`).

15. **폰트 로딩** (Pretendard/Google Fonts 선택 시): `layouts/partials/extend_head.html`
    ```html
    <link rel="preconnect" href="https://cdn.jsdelivr.net" crossorigin>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css">
    ```
    (System 선택 시 이 단계 스킵)

16. **로컬 검증**: `hugo server -D` → http://localhost:1313 — 메인/글/About 모두 렌더, 색상/폰트 적용 확인.

### Phase E — 배포 파이프라인

17. **GitHub repo 생성** (사용자 단계, 체크리스트):
    - User site → repo 이름 `<username>.github.io`
    - Project site → 임의 이름(예: `myblog`)

18. **`.github/workflows/hugo.yml`** (Hugo 공식 GitHub Pages 템플릿 기반):
    ```yaml
    name: Deploy Hugo site to Pages
    on:
      push:
        branches: [main]
      workflow_dispatch:
    permissions:
      contents: read
      pages: write
      id-token: write
    concurrency:
      group: "pages"
      cancel-in-progress: false
    defaults:
      run:
        shell: bash
    jobs:
      build:
        runs-on: ubuntu-latest
        env:
          HUGO_VERSION: 0.131.0
        steps:
          - name: Install Hugo CLI
            run: |
              wget -O ${{ runner.temp }}/hugo.deb \
                https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
              && sudo dpkg -i ${{ runner.temp }}/hugo.deb
          - name: Checkout
            uses: actions/checkout@v4
            with:
              submodules: recursive
              fetch-depth: 0
          - name: Setup Pages
            id: pages
            uses: actions/configure-pages@v5
          - name: Build with Hugo
            env:
              HUGO_ENVIRONMENT: production
              HUGO_ENV: production
            run: |
              hugo \
                --gc \
                --minify \
                --baseURL "${{ steps.pages.outputs.base_url }}/"
          - name: Upload artifact
            uses: actions/upload-pages-artifact@v3
            with:
              path: ./public
      deploy:
        environment:
          name: github-pages
          url: ${{ steps.deployment.outputs.page_url }}
        runs-on: ubuntu-latest
        needs: build
        steps:
          - name: Deploy to GitHub Pages
            id: deployment
            uses: actions/deploy-pages@v4
    ```
    **baseURL 단일 source-of-truth**: Actions가 `--baseURL`로 주입. `hugo.toml`의 `baseURL`은 로컬 개발용 fallback이며 빌드 시 CLI 인자가 우선함. 두 값은 일치해야 하나 충돌 시 CLI가 이김.

19. **Repo 연결 + push** (사용자 단계, 체크리스트):
    ```bash
    git remote add origin git@github.com:<username>/<repo>.git
    git add -A && git commit -m "init: hugo papermod blog"
    git push -u origin main
    ```

20. **GitHub Pages 활성화** (사용자 단계, 체크리스트):
    - Settings → Pages → Source = `GitHub Actions`

21. **Actions 검증**: Actions 탭에서 첫 빌드 성공 확인 후 배포 URL 접속.

### Phase F — 검증 & 마이그레이션 가이드

22. **`docs/migration.md`** (또는 README 섹션) — pandoc 기반:
    ```bash
    # Confluence에서 페이지 export → HTML 다운로드
    # 변환:
    pandoc -f html -t gfm-raw_html input.html -o content/posts/<slug>.md
    # 후처리:
    # - front matter 추가 (위 템플릿 복사)
    # - 코드블록 언어 힌트(```python 등) 보완
    # - 이미지를 static/images/<slug>/ 로 이동, 경로 수정
    ```
    수기 복붙은 **짧은 글의 fallback**으로만 명시.

23. **Rollback 가이드** (한 줄): "배포 후 깨짐 발견 시 `git revert <commit> && git push` — Actions가 이전 상태로 재배포."

24. **Draft-leak 방지**: 글 작성 중에는 front matter에 `draft: true`. Hugo는 기본적으로 `--buildDrafts` 없이 빌드하므로 자동 차단됨. README에 1줄 명시.

25. **최종 체크**: 위 Acceptance Criteria 16개 항목 검증.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Hugo 미설치/버전 불일치 | M | M | Step 5에서 `hugo version` 점검, `brew install hugo` 안내 |
| Repo-name 컨벤션 실수 | M | H | Phase A에서 deployment 모드 명시 선택, baseURL 분기 |
| PaperMod submodule이 clone 시 누락 | H | H | Actions `submodules: recursive` 명시, README에 `git clone --recurse-submodules` 안내 |
| PaperMod main drift로 빌드 깨짐 | M | M | **태그 핀(v8.x)**. README에 업그레이드 의식: `cd themes/PaperMod && git fetch --tags && git checkout vX.Y` |
| GitHub Pages Source를 Actions로 안 바꿔서 404 | H | H | Phase E 체크리스트에 명시, 첫 push 후 사용자 확인 단계 |
| 한글 폰트 미적용 | M | M | Phase A에서 폰트 전략 명시 선택, Pretendard fallback 체인 |
| baseURL 충돌 | L | M | Actions `--baseURL` 우선 + config의 fallback이 일치하도록 Phase A 입력값 사용 |
| 한글 슬러그 인코딩 문제 | L | M | front matter `slug: english-kebab` 권장, 가이드에 명시 |
| 마이그레이션 시 코드블록/표 깨짐 | M | L | pandoc 사용 + 후처리 체크리스트 |
| 커스터마이징 스코프 폭주 | M | M | **CSS ≤ 50줄 + partial 2개 한계** 강제 |

## Verification Steps

1. **Local build**: `hugo --gc --minify` → 0 에러, `public/index.html` 존재
2. **로컬 미리보기**: `hugo server -D` → http://localhost:1313, 메인/글/About 모두 진입
3. **모바일 반응형**: dev tools 375px, 가로 스크롤 0
4. **테마 커스터마이징**: dev tools로 `--primary`, `font-family` 사용자 값 확인
5. **CSS 천장**: `wc -l < assets/css/extended/custom.css` ≤ 50
6. **Submodule pin**: `git -C themes/PaperMod describe --tags --exact-match` 성공
7. **CI 빌드**: GitHub Actions Run 녹색 체크
8. **Production URL**: 배포 URL 200 OK, 글 진입, 다크모드/태그/검색 동작
9. **First-impression**: 30초 안에 이름+직무+글 목록 시각 확인 (non-blocking)

---

## ADR — Architecture Decision Record

### Decision
Hugo + PaperMod (태그 핀) + GitHub Pages (User site 권장) + GitHub Actions (Hugo 공식 워크플로 템플릿) + pandoc 기반 마이그레이션 가이드.

### Decision Drivers
1. 시간 예산 = 주말 (가장 빠른 배포)
2. 이직용 first impression (PaperMod의 검증된 디자인)
3. 0원 운영 (커스텀 도메인 없음, GitHub Pages 무료)
4. 사용자 명시 선택 (Hugo + PaperMod + GitHub Pages)
5. Go 친숙도 (저장소·CLI 경험 자산)

### Alternatives Considered
- **Astro + Cloudflare Pages**: Architect 스틸맨 후보. npm semver 테마, MDX, preview deploy. **Invalidated**: 사용자가 GitHub Pages 명시 선택. 단 Architect의 우려(테마 라이프사이클·마이그레이션 도구)는 v8.x 태그 핀과 pandoc 도입으로 흡수.
- **Hugo + Netlify free tier**: 동급 DX, 무료. **Invalidated**: GitHub Pages 명시 선택.
- **AWS S3 + hugo deploy**: 0원 원칙 위반. **Invalidated**.
- **Jekyll + Chirpy**: GitHub Pages 기본 지원. **Invalidated**: 사용자가 Hugo 선택, Ruby 환경 부담 회피.

### Why Chosen
사용자 제약(GitHub Pages, 0원, Hugo, PaperMod)을 모두 만족하는 유일한 옵션. Architect/Critic 우려사항은 다음으로 흡수:
- 태그 핀으로 PaperMod 라이프사이클 안정화
- Hugo 공식 Pages 워크플로(Actions가 baseURL 주입)로 double-injection 해소
- pandoc 도입으로 마이그레이션 ergonomics 개선
- 커스터마이징 천장(원칙 6) 명시로 원칙 1·5 긴장 해소

### Consequences
**긍정**:
- 주말 안에 배포 가능
- 유지비 0원, push만으로 배포
- PaperMod 기본 기능(다크모드/태그/검색/RSS) 즉시 활용
- 글 추가 = 마크다운 1개 생성 + push

**부정/제약**:
- 댓글·SEO 고도화·커스텀 도메인은 v2로 미룸 (의도적)
- PaperMod 업그레이드는 수동 의식 (README 명시됨)
- 디자인이 PaperMod 색채를 벗어나지 못함 (의도적, 천장 원칙)

### Follow-ups (v2 후보)
- giscus 댓글 (GitHub Discussions 기반)
- 커스텀 도메인 + Cloudflare proxy
- `/projects` 섹션 정식 분리
- 한국어 OG 이미지 자동 생성
- Confluence 자동 export 스크립트 (글 50개 넘어가면)
- Lighthouse 성능 ≥ 95점 가드

---

## Changelog (v1 → v2 적용된 개선)

**Architect 피드백**:
- ✅ Submodule 태그 핀 (Step 9)
- ✅ baseURL 단일 source-of-truth — Actions 우선 (Step 18 워크플로)
- ✅ Repo-name 분기 (Phase A 질문 2번)
- ✅ pandoc 마이그레이션 가이드 (Step 22)
- ✅ 커스터마이징 천장 — 원칙 6 신설 (CSS ≤ 50줄, partial 2개 한계)

**Critic 피드백**:
- ✅ Acceptance Criteria `[automated]` vs `[user-confirm]` 마킹
- ✅ Phase A에 폰트 전략 질문 추가 (System/Pretendard/Google)
- ✅ `.omc/` 보호 코멘트 (Step 6)
- ✅ Step 18에서 정확한 Actions 버전 명시 (`actions/checkout@v4`, `actions/configure-pages@v5`, `actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4`)
- ✅ Rollback 가이드 (Step 23)
- ✅ Draft-leak 방지 노트 (Step 24)
- ✅ `enableRobotsTXT = true` (Step 10 hugo.toml)
- ✅ Option C(Hugo+Netlify) 추가 invalidation으로 alternative depth 보강

**의도적으로 제외**:
- LICENSE, Image strategy(LFS): 1차 출시 범위 밖. v2 follow-up에 포함.
- Lighthouse 자동화: v2.

## Open Questions for Execution (Phase A에서 확정)
- GitHub username
- Deployment 모드 (User site 권장 vs Project site)
- 메인 색상 (hex 또는 톤)
- 폰트 전략 (System/Pretendard/Google)
- 첫 게시글 콘텐츠 (선택, 없으면 placeholder)
- 사이트 제목/슬로건
