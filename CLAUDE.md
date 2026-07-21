# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Firefly is a feature-rich static blog theme built on **Astro 6** with **Svelte 5** for interactive components. It's a fork of [Fuwari](https://github.com/saicaca/fuwari) extended with extensive features. Primary language is Chinese (Simplified) with i18n for en, zh_TW, ja, ru.

## Commands

| Command | Purpose |
|---|---|
| `pnpm dev` | Dev server at `localhost:4321` |
| `pnpm build` | Production build (icons → LQIPs → Astro build → Pagefind indexing) |
| `pnpm preview` | Preview production build |
| `pnpm check` | `astro check` for type/error checking |
| `pnpm type-check` | `tsc --noEmit --isolatedDeclarations` |
| `pnpm lint` | Biome lint + auto-fix |
| `pnpm format` | Biome format |
| `pnpm new-post <filename>` | Scaffold a new blog post |

Package manager is **pnpm** (enforced). Node.js >= 22 required.

## Architecture

### Astro + Svelte Hybrid

- `.astro` components for static content and layouts
- `.svelte` components for interactive UI (search, settings, pagination, archive) — mounted with `client:load` or `client:visible`
- Swup.js handles SPA-like page transitions with multiple container targets

### Configuration-Driven

All features are toggled/configured via TypeScript files in `src/config/`, exported through the barrel at `src/config/index.ts`. Key configs:

- `siteConfig.ts` — core site settings, theme, pagination
- `sidebarConfig.ts` — sidebar layout (left/right/both, widget ordering)
- `commentConfig.ts`, `analyticsConfig.ts`, `fontConfig.ts`, etc.

### Layout System

- `Layout.astro` — base HTML shell (head, body, theme init, analytics, Swup hooks)
- `MainGridLayout.astro` — full page grid with sidebar(s), navbar, wallpaper, footer

### Content Collections

Defined in `src/content.config.ts`:
- `posts` — blog posts (`.md`/`.mdx`) with frontmatter: title, published, tags, category, draft, pinned, password, comment, etc.
- `spec` — special pages (about, guestbook)

### Key Directories

- `src/components/` — organized by domain: `analytics/`, `comment/`, `common/`, `controls/`, `features/`, `layout/`, `misc/`, `pages/`, `widget/`
- `src/plugins/` — 15 custom remark/rehype plugins (Mermaid, PlantUML, KaTeX, GitHub cards, reading time, etc.)
- `src/i18n/` — translation keys in `i18nKey.ts`, language files in `languages/*.ts`, lookup via `translation.ts`
- `src/utils/` — content sorting, crypto (encrypted posts), date formatting, image processing/LQIP, TOC generation
- `src/pages/` — Astro file-based routing
- `scripts/` — build-time utilities (`generate-icons.js`, `generate-lqips.ts`, `new-post.js`)

### Path Aliases (tsconfig.json)

`@components/*`, `@assets/*`, `@constants/*`, `@utils/*`, `@i18n/*`, `@layouts/*` → `./src/<dir>/*`; `@/*` → `./src/*`

## Code Style

- **Biome** enforces: tab indentation, double quotes, recommended lint rules
- Relaxed rules for `.svelte`/`.astro` files (useConst off, noUnusedVariables off)
- Commit convention: **Conventional Commits** (`feat:`, `fix:`, `chore:`, etc.)

## Build Pipeline

Multi-step: `scripts/generate-icons.js` → `scripts/generate-lqips.ts` → `astro build` → `pagefind --site dist`

Icons/LQIP data are generated into `src/constants/` and committed. Regenerate with `pnpm icons` or `pnpm lqips`.

## Deployment

- **Vercel** (default, `vercel.json`)
- **Cloudflare Workers** (`wrangler.jsonc`, set `CF_WORKERS` env var)
- Static output to `dist/`

## ⚠️ 敏感信息安全规则

该项目为**公开仓库**，推送到 GitHub 前必须遵守以下规则：

### 禁止提交到 git 的敏感信息

| 类型 | 示例 | 处理方式 |
|------|------|---------|
| 服务器 IP 地址 | `YOUR_SERVER_IP` | 用 `YOUR_SERVER_IP` 或变量替代 |
| SSH 私钥路径 | `~/.ssh/YOUR_SSH_KEY` | 用 `~/.ssh/YOUR_KEY` 替代 |
| SSH 用户名 | `root@server` | 用 `YOUR_USER@YOUR_SERVER` 替代 |
| 内网段 CIDR | `10.0.0.0/24` | 删除或泛化为 `内网段` |
| API 密钥/Token | `sk-xxx`, `ghp_xxx` | 使用环境变量或 GitHub Secrets |
| 私钥/证书内容 | `-----BEGIN PRIVATE KEY-----` | 绝不可提交 |
| 服务器内部路径 | `/root/cc/...` | 用 `/path/to/...` 替代 |
| `archive/deploy/` 文件 | README, nginx.conf | 已去敏处理，勿再添加敏感 IP |

### 推送前检查清单

1. **gitleaks 扫描**（推荐）:
   ```bash
   gitleaks detect --source . -c .gitleaks.toml -v
   ```
2. **手动检查**含敏感模式的文件:
   - `archive/deploy/*` — 是否引入了新的 IP 或凭证
   - `server/nginx/*.template` — 注释中是否有真实 IP/网段
   - 所有 `.md` 文件 — 博客文章中的 IP 必须用 `xx.xx.xx.xx`
3. **检查环境变量**:
   - `.env` 文件绝不可提交（已在 `.gitignore`）
   - 所有 Secrets 值必须用 `${{ secrets.* }}` 引用
4. **检查配置文件注释**:
   - 注释中的 IP、路径、凭证示例必须是占位符

### 安全配置规范

- **服务器部署配置**: 使用 `envsubst` 模板 + GitHub Secrets，不在仓库中留明文
- **Workflow 文件**: 所有敏感值通过 `${{ secrets.* }}` 引用
- **博客文章**: 涉及安全事件/服务器时，IP 用 `xx.xx.xx.xx` 替代
- **GitHub Secrets 需要配置**（在 repo → Settings → Secrets and variables → Actions）:
  - `SERVER_HOST` — 服务器 IP
  - `SERVER_USERNAME` — SSH 用户名
  - `SERVER_SSH_KEY` — SSH 私钥
  - `BLOG_DOMAIN` — 博客域名
  - `DEV_MACHINE_IP` — 开发机 IP 白名单

## Agent skills

### Issue tracker

Issues and PRDs for this repo live as GitHub issues. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles use their default names. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context repo. See `docs/agents/domain.md`.

