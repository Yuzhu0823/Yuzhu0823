# Yuzhu — personal blog

A minimal Jekyll site: Home / Blog / Gallery, in the same quiet, typography-first spirit as the original Wix version.

## Run locally

```bash
gem install bundler
bundle install
bundle exec jekyll serve
```

Then open http://localhost:4000.

## Publish on GitHub Pages

1. Create a new GitHub repo (e.g. `yuzhu-blog`) and push this folder's contents to it.
2. In the repo, go to **Settings → Pages**, set source to the `main` branch (root).
3. If the repo is *not* named `<your-username>.github.io`, edit `_config.yml` and set `baseurl: "/yuzhu-blog"` (or whatever the repo is called) and `url` to `https://<your-username>.github.io`.
4. Push — GitHub builds and deploys automatically in a minute or two.

## Add a new post

Create a file in `_posts/` named `YYYY-MM-DD-title.md`:

```markdown
---
title: "My New Post"
---

Post content in Markdown goes here.
```

## Add gallery images

Drop image files into `assets/img/` and add a matching `<figure>` block in `gallery/index.md`.
