# eunbc.github.io

[Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (v8.0) 기반 개인 블로그.
GitHub Actions로 자동 배포 → https://eunbc.github.io/

## 빠른 시작

```bash
# 클론 시 submodule(테마)까지 함께
git clone --recurse-submodules git@github.com:eunbc/eunbc.github.io.git
cd eunbc.github.io

# 로컬 미리보기 (draft 포함)
hugo server -D
# → http://localhost:1313
```

## 글 추가

```bash
hugo new content posts/my-new-post.md
# → content/posts/my-new-post.md 가 archetypes 템플릿으로 생성됨
# 작성 중에는 front matter `draft: true` 유지, 발행 시 `false`로 변경
```

front matter 표준 템플릿:

```yaml
---
title: "글 제목"
date: 2026-05-09T10:00:00+09:00
draft: false
tags: ["tag1", "tag2"]
summary: "한 줄 요약 (목록·OG에서 사용)"
slug: "english-kebab-slug"   # 한글 슬러그 인코딩 회피
---
```

## 배포

```bash
git push origin main
# → GitHub Actions가 빌드/배포. Actions 탭에서 진행 상황 확인.
```

배포 URL: https://eunbc.github.io/

## 디자인 커스터마이징

**스코프 천장 (의도적으로 좁게 유지)**:

- `assets/css/extended/custom.css` — 색상·폰트 오버라이드 (≤ 50줄)
- `layouts/partials/extend_head.html` — 외부 폰트/스크립트
- `layouts/partials/extend_footer.html` — (필요 시)
- 그 외 `layouts/_default/*` 등 깊은 오버라이드는 **금지**

이 천장을 넘어야 한다면 v2로 미루세요.

## PaperMod 업그레이드 의식

테마는 `v8.0`에 핀(pin) 되어 있습니다. 업그레이드 시:

```bash
cd themes/PaperMod
git fetch --tags
git tag -l 'v*' | sort -V | tail -5      # 최신 태그 확인
git checkout vX.Y                         # 원하는 태그로 이동
cd ../..
git add themes/PaperMod
git commit -m "chore: upgrade PaperMod to vX.Y"
```

CHANGELOG: https://github.com/adityatelange/hugo-PaperMod/releases

## Confluence 마이그레이션

[`docs/migration.md`](./docs/migration.md) 참조. 요약:

```bash
# Confluence에서 페이지 → ⋯ → Export → HTML로 다운로드
pandoc -f html -t gfm-raw_html input.html -o content/posts/<slug>.md
# 후처리: front matter 추가, 코드블록 언어, 이미지 경로
```

## 롤백

배포 후 깨짐 발견 시:

```bash
git revert <commit-sha> && git push
# → Actions가 이전 상태로 재배포
```

## 라이선스

콘텐츠: 별도 표기 전까지 © eunbc, all rights reserved.
코드(테마 제외): MIT 권장 (필요 시 LICENSE 파일 추가).
