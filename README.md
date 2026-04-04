# hexo-theme-pretext

简洁日式排版风格的 Hexo 博客主题，集成 [@chenglou/pretext](https://github.com/chenglou/pretext) ASCII 动画效果。

A minimal Japanese-inspired Hexo blog theme with ASCII animations powered by [@chenglou/pretext](https://github.com/chenglou/pretext).

---

## Features

- Clean, spacious Japanese typography with generous whitespace
- Dark mode (auto + manual toggle)
- ASCII animations via `@chenglou/pretext` (progressive enhancement)
- Responsive, mobile-first design
- Multiple comment systems (Giscus, Disqus, Waline)
- i18n support (zh-CN, en)
- RSS autodiscovery
- Code copy buttons
- Table of Contents

## Prerequisites

- [Hexo](https://hexo.io/) >= 5.0
- [hexo-renderer-ejs](https://github.com/hexojs/hexo-renderer-ejs) >= 2.0

## Installation / 安装

### Via npm (recommended)

```bash
npm install hexo-theme-pretext hexo-renderer-ejs
```

Edit your site `_config.yml`:

```yaml
theme: pretext
```

### Via git

```bash
git clone https://github.com/hexo-theme-pretext/hexo-theme-pretext.git themes/pretext
npm install hexo-renderer-ejs
```

Edit your site `_config.yml`:

```yaml
theme: pretext
```

## Configuration / 配置

Create `_config.pretext.yml` in your site root (Hexo 5+), or edit `themes/pretext/_config.yml`.

```yaml
# Navigation
menu:
  Home: /
  Archives: /archives
  Tags: /tags
  About: /about

# Profile
profile:
  name: "Your Name"
  avatar: /images/avatar.png
  bio: "A short bio"
  social:
    github: your-username
    twitter: your-handle
    email: you@example.com

# Pretext ASCII animations
pretext:
  hero:
    enable: true
    text: "Your Blog Title"
    animation: "wave"       # wave | breathe | glitch
  post_ornament:
    enable: true
    style: "geometric"      # geometric | dots | lines
  archive_flow:
    enable: true
  page_404:
    enable: true
    art: "cat"              # cat | mountain | maze

# Comments
comments:
  provider: false           # giscus | disqus | waline | false
  giscus:
    repo: ""
    repo_id: ""
    category: ""
    category_id: ""
  disqus:
    shortname: ""
  waline:
    server_url: ""

# Features
features:
  toc: true
  reading_time: true
  code_copy: true
  back_to_top: true
```

See `_config.yml` in the theme directory for all available options.

## Pretext Integration

Pretext is used as a **progressive enhancement layer** — all content is fully readable without JavaScript. The four animation scenes (hero, post ornament, archive flow, 404 easter egg) load `@chenglou/pretext` from CDN on demand and gracefully degrade to CSS fallbacks.

## Disqus Support

To enable Disqus, set `disqus_shortname` in your site's `_config.yml`, or configure via the theme's comment settings:

```yaml
# Site _config.yml
disqus_shortname: your-shortname

# Or theme _config.pretext.yml
comments:
  provider: disqus
  disqus:
    shortname: your-shortname
```

## License

[MIT](LICENSE)

Theme by hexo-theme-pretext contributors.
[Pretext](https://github.com/chenglou/pretext) (MIT) by Cheng Lou.
