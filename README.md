# dvlog

DBA로 일하면서 배운 것들을 기록하는 개인 기술 블로그입니다. 주로 데이터베이스(PostgreSQL, MySQL 등) 관련 내용을 다룹니다.

## 기술 스택

- **[Astro 5](https://astro.build/)** — content collections 기반 정적 사이트
- **View Transitions API** (`<ClientRouter />`) — 풀 리로드 없는 부드러운 페이지 전환
- 손으로 작성한 CSS + CSS 변수 디자인 토큰 (라이트/다크 모드)
- 코드 하이라이팅: Astro 내장 Shiki (`github-light` / `github-dark`)
- 폰트: Pretendard (본문/제목), IBM Plex Mono (코드)
- RSS(`/rss.xml`), sitemap, OG 메타 태그

## 개발

```bash
npm install      # 의존성 설치
npm run dev      # 개발 서버 (http://localhost:4321)
npm run build    # 프로덕션 빌드 → ./dist
npm run preview  # 빌드 결과 미리보기
```

## 글 작성

`src/content/` 아래에 마크다운 파일을 추가합니다. frontmatter 예시:

```yaml
---
title: 글 제목
description: 한 줄 요약
pubDate: 2025-03-12
tags:
  - postgresql
  - db/lock
---
```

이미지는 `public/images/`에 두고 본문에서 `/images/파일명.png`로 참조합니다.

## 배포

`main` 브랜치에 push하면 GitHub Actions가 자동으로 빌드해 GitHub Pages(<https://okyungjin.github.io>)에 배포합니다.

> GitHub 저장소 설정 → **Settings → Pages → Build and deployment → Source**를 **GitHub Actions**로 설정해야 합니다.
