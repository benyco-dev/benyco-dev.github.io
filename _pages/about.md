---
permalink: /
title: "Benyco's Blog"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

클라우드와 인프라를 직접 만들어보고 남기는 기록입니다.

문서를 요약하는 대신, 실제로 만들어보다 막혔던 지점과 그때 왜 그렇게 정했는지를 적습니다.

## 다루는 것

- **클라우드 인프라** — Google Cloud (Compute Engine, Cloud Run, IAM, 네트워크)
- **인프라 코드(IaC)** — Terraform으로 VPC, VM 같은 GCP 리소스를 코드로 만들고 지우기
- **컨테이너와 배포** — Docker 이미지 작성, GitHub Actions 빌드·배포 자동화
- **사이드 프로젝트** — 만들면서 나온 실패와 측정 결과 위주로

## 최근 글

**[Auto Trading 시리즈](/categories/#auto-trading)** — Claude Code와 토스증권 API로 미국 주식 자동매매를 만든 기록

- [1편 환경 구성](/auto-trading-1-setup/) — Claude Code 바이브 코딩, 토스증권 Open API, GCP 서버 구성
- [2편 수수료의 함정](/auto-trading-2-commission/) — 백테스트는 22%였는데 수수료를 넣으니 4%가 됐다
- [3편 마무리](/auto-trading-3-wrapup/) — 백테스트 숫자를 어디까지 믿을 것인가

**Google Cloud**

- [Compute Engine에 Docker로 Nginx 띄우기](/gcp-docker-nginx/)
- [Cloud Run에 Docker 이미지 배포하기](/cloud-run-docker-gui/)

**[Terraform](/categories/#terraform)**

- [Terraform으로 GCP에 VPC + VM 만들고 지워보기](/terraform-gcp-vpc-vm/)

[전체 글 보기](/year-archive/) · [GitHub](https://github.com/benyco-dev)
