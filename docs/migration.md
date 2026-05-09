# Confluence → 블로그 마이그레이션 가이드

이 블로그는 양보다 질을 우선합니다. **자랑스러운 글 5–10개만** 골라서 옮기세요.

## 권장: Pandoc 일괄 변환

### 1. Confluence에서 export

1. 옮길 페이지로 이동
2. 우측 상단 `⋯` (또는 `Tools` 메뉴) → `Export` → `Export to HTML`
3. 다운로드된 zip을 풀고 `index.html` (또는 페이지명.html) 위치 확인

### 2. Pandoc 설치 (없다면)

```bash
brew install pandoc
```

### 3. 변환

```bash
pandoc -f html -t gfm-raw_html \
  ~/Downloads/Confluence-export/index.html \
  -o content/posts/my-imported-post.md
```

옵션 설명:
- `-f html`: 입력은 HTML
- `-t gfm-raw_html`: GitHub-flavored Markdown으로, raw HTML은 가능한 제거
- 결과는 `content/posts/`에 저장

### 4. 후처리 체크리스트

변환된 마크다운을 열고 아래를 확인하세요.

- [ ] **front matter 추가** (파일 맨 위에):
  ```yaml
  ---
  title: "원래 페이지 제목"
  date: 2024-XX-XX     # Confluence 작성일로
  draft: false
  tags: ["tag1", "tag2"]
  summary: "한 줄 요약"
  slug: "english-kebab"
  ---
  ```
- [ ] **첫 줄의 `# 제목` 제거** — Hugo가 front matter title을 사용함
- [ ] **코드블록 언어 힌트**: ` ``` ` → ` ```python `, ` ```bash ` 등 명시
- [ ] **이미지 이동**:
  - export zip 안의 `attachments/` 또는 `images/` → `static/images/<slug>/` 로 복사
  - 마크다운 안의 경로 수정: `![](attachments/foo.png)` → `![](/images/<slug>/foo.png)`
- [ ] **Confluence 매크로 잔재 정리** (info, panel, expand 등 → 일반 텍스트나 blockquote로)
- [ ] **표 깨짐 확인** — 가끔 pandoc이 어색하게 변환함, 수기 정리
- [ ] **로컬 미리보기**: `hugo server -D`로 렌더링 확인

## 짧은 글의 fallback: 수기 복붙

200줄 미만의 글은 그냥 마크다운 에디터에서 손으로 옮기는 게 빠를 수 있어요.

1. Confluence에서 본문 영역 드래그 → 복사
2. VS Code (또는 기타 에디터)에 붙여넣기 → 마크다운으로 정리
3. 위 후처리 체크리스트 동일 적용

## 마이그레이션 우선순위 가이드

| 점수 기준 | 의미 |
|----------|------|
| 5점 | "이 글로 채용 면접 들어가도 자신 있다" |
| 4점 | 잘 정리된 기술 노트 |
| 3점 | 쓸만하지만 약간의 정리 필요 |
| ≤2점 | 옮기지 않음 |

**4–5점만 옮기세요.** 5–10개로 시작해서 점진적으로 추가하면 됩니다.

## 한국어 슬러그 주의

`slug` 필드는 반드시 영문 kebab-case로:

```yaml
slug: "kafka-consumer-rebalance-deep-dive"   # OK
slug: "카프카-컨슈머-리밸런스"                  # 안 됨 (URL 인코딩 문제)
```
