# 따분한녀석의 고민 - 기술 블로그

ChoDaeYun의 개인 기술 블로그입니다.

## 📝 블로그 주소

https://chodaeyun.github.io

## 🛠️ 기술 스택

- **Jekyll**: 정적 사이트 생성기
- **Texture Theme**: 미니멀한 블로그 테마
- **GitHub Pages**: 호스팅

## 🚀 로컬 실행 방법

### 1. 사전 요구사항

- Ruby 3.1.0 이상
- rbenv (Ruby 버전 관리)
- Bundler

### 2. Ruby 버전 설정

```bash
# rbenv 설치 (Mac)
brew install rbenv

# Ruby 3.1.0 설치
rbenv install 3.1.0

# 프로젝트 디렉토리에서 Ruby 버전 설정 (이미 .ruby-version 파일 존재)
rbenv local 3.1.0
```

### 3. 의존성 설치

```bash
# rbenv 초기화
eval "$(rbenv init - zsh)"

# Bundle 설치
bundle install
```

### 4. 서버 실행

```bash
# Jekyll 서버 시작
eval "$(rbenv init - zsh)" && bundle exec jekyll serve

# 또는 간단하게
bundle exec jekyll serve
```

서버가 실행되면 http://127.0.0.1:4000 에서 확인할 수 있습니다.

### 5. 서버 중지

터미널에서 `Ctrl + C` 를 누르면 서버가 중지됩니다.

## 📂 프로젝트 구조

```
ChoDaeYun.github.io/
├── _config.yml          # Jekyll 설정 파일
├── _posts/              # 블로그 포스트 (markdown)
├── _layouts/            # 페이지 레이아웃
├── _includes/           # 재사용 가능한 컴포넌트
├── _sass/               # SCSS 스타일
├── assets/              # 이미지, CSS, JS 파일
├── _site/               # 생성된 정적 파일 (git 무시)
├── Gemfile              # Ruby 의존성
└── README.md            # 이 파일
```

## ✍️ 새 글 작성하기

1. `_posts/` 디렉토리에 파일 생성
2. 파일명 형식: `YYYY-MM-DD-제목.markdown`
3. Front Matter 작성:

```markdown
---
layout: post
title:  "글 제목"
description: 짧은 설명
date:   YYYY-MM-DD HH:MM:SS +0000
categories: 카테고리1 카테고리2
---

본문 내용...
```

## 🔧 문제 해결

### Ruby 버전 오류

```bash
# rbenv 초기화 후 다시 시도
eval "$(rbenv init - zsh)"
bundle install
```

### Jekyll 명령어를 찾을 수 없음

```bash
# Bundle을 통해 실행
bundle exec jekyll serve
```

### 포트가 이미 사용 중

```bash
# 다른 포트로 실행
bundle exec jekyll serve --port 4001
```

## 📚 시리즈

### 모아갤 프로젝트 (2026.03~)
- Claude와 함께하는 Flutter 앱 개발 여정
- 프로젝트 시작부터 배포까지의 과정 기록

### 인프라 & DevOps
- Kubernetes, CI/CD, 모니터링 관련 글

### 백엔드 개발
- Spring, Redis, DB 관련 경험 공유

## 📮 연락처

- GitHub: [@ChoDaeYun](https://github.com/ChoDaeYun)
- Email: chody0116@gmail.com

## 📄 라이선스

이 블로그의 모든 글은 개인 저작물이며, 무단 전재를 금지합니다.
