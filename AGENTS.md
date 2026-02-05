# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog/website at [guangda.me](https://guangda.me), built with Hugo using the PaperMod theme (dark mode). Deployed to GitHub Pages via GitHub Actions.

## Commands

- **Setup** (first time after clone): `hugo mod get`
- **Dev server**: `hugo server -D` (includes drafts)
- **Build**: `hugo --minify`
- **New post**: `hugo new posts/<slug>.md`

## Architecture

- **Hugo version**: 0.155.2 (extended), pinned in `.github/workflows/gh-pages.yml`
- **Theme**: PaperMod, installed via Hugo Modules (no git submodule)
- **Content**: Markdown files in `content/` using `+++` TOML front matter (not `---` YAML)
- **Projects list**: `content/projects.md` — a standalone markdown page listing projects
- **Static assets**: `static/images/` for images, `static/CNAME` for custom domain
- **Deployment**: Pushing to `main` triggers a GitHub Actions workflow that builds with Hugo and deploys directly to GitHub Pages (no `gh-pages` branch). Source: `actions/configure-pages` + `actions/deploy-pages`.

## Content Conventions

- Posts use TOML front matter with `title`, `author`, `date`, `lastmod`, `categories`, `draft` fields
- The `about.md` page lives at `content/about.md` (not in posts)
- Date format: `2006-01-02` (Go reference time)
- Code highlighting: Dracula style with line numbers
