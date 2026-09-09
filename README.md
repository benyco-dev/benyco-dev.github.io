# Benyco's Blog

[benyco-dev.github.io](https://benyco-dev.github.io)

클라우드와 인프라를 직접 만들어보고 남기는 기록. Google Cloud, Docker, 자동매매 프로젝트를 다룬다.

Jekyll + GitHub Pages로 운영한다.

## 글 쓰기

`_posts/`에 `YYYY-MM-DD-slug.md` 형식으로 추가한다. URL은 파일명의 slug 부분으로 정해진다
(`_config.yml`의 `permalink: /:title/`).

```markdown
---
title: "글 제목"
date: 2026-09-10
categories:
  - Auto Trading
tags:
  - GCP
  - Docker
excerpt: 검색 결과와 목록에 노출되는 한 줄 요약.
---

본문
```

`excerpt`는 넣는 편이 좋다. 없으면 본문 첫 줄이 잘려서 검색 결과에 나온다.

초안은 `_drafts/`에 두면 사이트에도, 저장소에도 올라가지 않는다 (gitignore 처리됨).

## 로컬에서 미리 보기

```bash
bundle install
bundle exec jekyll serve
```

http://localhost:4000 에서 확인한다. Ruby 환경을 깔기 싫으면 Docker로도 된다.

```bash
docker compose up
```

## 배포

`master`에 push하면 GitHub Pages가 자동으로 빌드해서 배포한다. 별도 작업 없음.

## 구조

| 경로 | 용도 |
|---|---|
| `_posts/` | 발행된 글 |
| `_drafts/` | 초안 (커밋되지 않음) |
| `_pages/` | 고정 페이지 (About 등) |
| `_config.yml` | 사이트 설정, SEO, 메뉴 |
| `images/`, `files/` | 이미지와 첨부 파일 |

## 라이선스

테마 코드는 [Academic Pages](https://github.com/academicpages/academicpages.github.io)
(Michael Rose의 [Minimal Mistakes](https://github.com/mmistakes/minimal-mistakes) 기반)를
fork해서 쓰고 있고 MIT 라이선스를 따른다.

`_posts/`, `_pages/`, `images/`의 글과 이미지는 저작권 보유. 자세한 내용은 [LICENSE](LICENSE) 참고.
