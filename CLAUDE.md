# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hugo static site blog using the PaperMod theme, deployed to GitHub Pages at https://blog.enjoythejourney.xyz/. The blog is written primarily in Korean.

## Common Commands

```bash
# Run local development server
hugo server -D

# Build the site (output to ./public)
hugo

# Build for production (minified)
hugo --gc --minify

# Create a new post (draft by default)
hugo new content/tech/my-new-post.md
hugo new content/startup/my-new-post.md
hugo new content/life/my-new-post.md
hugo new content/projects/my-new-post.md
```

## Content Structure

- `content/about.md` - About page
- `content/tech/` - Technical articles (기술)
- `content/startup/` - Startup-related posts (스타트업)
- `content/life/` - Personal/lifestyle posts (일상)
- `content/projects/` - Project documentation (프로젝트)

## Configuration

- `config.toml` - Main Hugo configuration
- `archetypes/default.md` - Default front matter template for new posts
- Theme: PaperMod (as git submodule in `themes/PaperMod/`)

## Deployment

Automated via GitHub Actions (`.github/workflows/hugo.yml`):
- Triggers on push to `main` branch
- Uses Hugo extended v0.152.2
- Deploys to GitHub Pages

## Post Front Matter

New posts are created as drafts. To publish, set `draft = false` in the front matter:

```toml
+++
date = '2025-12-03'
draft = false
title = 'My Post Title'
+++
```

## Writing Style Guide (글쓰기 모드)

블로그 글 작성 시 다음 스타일을 따른다:

### 말투
- **해요체** 사용 (~합니다, ~입니다, ~습니다 대신 ~해요, ~이에요, ~어요)
- 친근하고 편안한 톤 유지
- 독자에게 직접 말하는 듯한 느낌 ("~하려 합니다" → "~하려 해요")

### 문체 특징
- 개인적인 경험과 생각을 자연스럽게 공유
- 적절한 곳에 **볼드체**로 강조
- `---` 구분선으로 섹션 구분
- 리스트와 헤딩을 활용한 가독성 높은 구조

### 예시
```
❌ "이 블로그에서는 기술 관련 글을 작성합니다."
✅ "이 블로그에서는 기술 관련 글을 작성하려 해요."

❌ "제가 경험한 내용을 공유하겠습니다."
✅ "제가 경험한 내용을 공유하려 해요."
```
