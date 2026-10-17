# fletcher.cool

Personal website / contracting portfolio for Fletcher (embedded systems engineer). Static site built with [Zola](https://www.getzola.org/documentation/) and the [tabi](https://github.com/welpo/tabi) theme, hosted on GitHub Pages at https://fletcher.cool.

## Structure

- `config.toml`: site config. Theme options live under `[extra]` (menu, socials, footer, CSP). See `themes/tabi/config.toml` for every available option with documentation.
- `content/_index.md`: home page (intro header, latest posts from `blog/`, featured projects from `projects/`).
- `content/blog/<slug>/index.md`: blog posts (page bundles; put images next to `index.md`). Front matter: `title`, `description`, `date`, `[taxonomies] tags`, `[extra] toc = true` for long posts.
- `content/projects/<slug>/index.md`: project cards, ordered by `weight`, rendered with tabi's `cards.html`.
- `content/pages/{about,services}/index.md`: standalone pages (`template = "info-page.html"`, `path = "<name>"` so they live at `/about/`, `/services/`). `content/pages/_index.md` has `render = false`.
- `static/`: copied verbatim to the site root. Standalone web apps/games go in e.g. `static/games/<name>/` and can be embedded in posts with an `<iframe>` (add the origin to `allowed_domains` in config if it's external).
- `themes/tabi`: git submodule pinned to a release tag (currently v5.0.0). Clone with `git clone --recurse-submodules`, or run `git submodule update --init`.

## Theme reference

The tabi demo content in `themes/tabi/content/` doubles as documentation, e.g. `blog/mastering-tabi-settings`, `blog/shortcodes`, `blog/comments`, `blog/markdown`. Read those for examples.

- tabi requires `[markdown.highlighting] style = "class"`.
- CSP is enabled (`enable_csp`), so inline scripts and external embeds are blocked unless their domains are added to `extra.allowed_domains`. `zola serve` injects an inline livereload script that the CSP blocks; this shows a console error in dev only and is harmless.
- Comments: giscus (GitHub Discussions on this repo, category "Announcements", mapped by pathname), configured in `[extra.giscus]` and on for every post (`enabled_for_all_posts`). Disable on one post with `giscus = false` under `[extra]`. Needs the giscus GitHub App installed on the repo and Discussions enabled. `extra.allowed_domains` is set explicitly (tabi's defaults plus `https://giscus.app` in style-src, for giscus's `default.css`); tabi adds giscus.app to script-src/frame-src itself.
- A Remark42 server was tried and is stopped: its files and data remain in `/home/pi/remark42/` on the Pi (pi@192.168.1.100), with a runbook in its README. Local branch `remark42-comments` has the matching site templates.

## Build and deploy

- Local preview: `zola serve` (http://127.0.0.1:1111). Build: `zola build` (output goes to `public/`, which is gitignored).
- Zola binary: v0.23.6 at `~/.local/bin/zola`. Keep it in sync with `zola_version` in `.github/workflows/deploy.yml`.
- Deploy: every push to `main` runs `.github/workflows/deploy.yml` (getzola/github-pages action, then actions/deploy-pages). The Pages source is set to "GitHub Actions". The custom domain is set in the repo's Pages settings, not with a CNAME file.

## DNS (Porkbun)

The domain `fletcher.cool` is registered at Porkbun. The apex has A records `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153` and AAAA records `2606:50c0:8000::153` through `2606:50c0:8003::153`. `www` is a CNAME to `fletcherd.github.io`.
