# In Base — Hugo site

The "FreeBSD Base: Things You Didn't Know It Could Do" blog series.
Static site built with [Hugo](https://gohugo.io) (extended, v0.163.3), deployed
to DigitalOcean App Platform's free static-site tier.

## Local development

Install Hugo **extended** (the CSS pipeline needs it):

```sh
# FreeBSD
pkg install hugo
# macOS
brew install hugo
# or grab the extended binary from https://github.com/gohugoio/hugo/releases
```

Then:

```sh
hugo server -D        # live-reload dev server at http://localhost:1313 (-D shows drafts)
hugo --gc --minify    # production build into ./public
```

## Writing a post

```sh
hugo new posts/my-post-title.md
```

The archetype pre-fills front matter, including `draft = true` and
`freebsd_version = "15.1"` (the version badge shown in the post meta and list).
Set `draft = false` to publish. Fill in `summary` — it's used for the post list
and the meta description.

Convention for these posts: open with a runnable one-liner, name the
Linux/macOS equivalent, cite the man page, end with "why base matters here."

## Deploying to DigitalOcean App Platform

1. Push this repo to GitHub.
2. Edit `.do/app.yaml`: set `github.repo` to your `username/repo`.
3. In the DO control panel: **Create App → GitHub → pick the repo.** It reads
   `.do/app.yaml` automatically. Or via CLI: `doctl apps create --spec .do/app.yaml`.
4. Every `git push` to `main` rebuilds and redeploys (`deploy_on_push: true`).

Static sites are free (up to 3 apps). The 1 GiB/mo egress allowance overflows
at $0.02/GiB — negligible, and covered by account credits.

### Custom domain

Add the domain in the DO panel, point your DNS at the provided records, then
uncomment the `domains:` block in `.do/app.yaml`. TLS is provisioned and renewed
automatically.

## Structure

```
content/posts/   the posts
content/about.md  about page
layouts/          templates (man-page-styled masthead, single, list, 404)
assets/css/       main.css (includes Chroma syntax highlighting, nord-derived)
archetypes/       front-matter template for `hugo new`
.do/app.yaml      App Platform deploy spec
hugo.toml         site config
```

## Theme notes

The look takes the FreeBSD man page as its visual ancestor: a monospace
`IN BASE(7)` masthead, a serif reading face, and dark code blocks as the
centerpiece. Colors live as CSS custom properties at the top of
`assets/css/main.css` (including a dark-mode block) — change `--beastie` to
re-accent the whole site. Syntax-highlight colors are the Chroma classes
appended below the main styles; regenerate a different scheme with
`hugo gen chromastyles --style=NAME`.
