# imkrupakar.github.io

Source for my blog at <https://imkrupakar.github.io>, built with [Jekyll](https://jekyllrb.com) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, deployed to GitHub Pages by
[GitHub Actions](.github/workflows/pages-deploy.yml) on every push to `main`.

## Writing a post

1. Create `_posts/YYYY-MM-DD-slug.md`. The post URL becomes `/posts/slug/`.
2. Add front matter:

   ```yaml
   ---
   title: "Post title"
   date: 2026-11-09 10:00:00 +0530
   categories: [Top Category, Sub Category]   # at most two levels
   tags: [tag-one, tag-two]                   # lowercase
   description: One-sentence summary used in listings and SEO.
   media_subpath: /assets/img/posts/slug      # optional, lets you write ![alt](file.png)
   image:                                     # optional cover / social preview
     path: cover.png
     alt: Cover description
   ---
   ```

3. Put images in `assets/img/posts/<slug>/`. An italic line directly under an image renders as its caption.
4. Work-in-progress posts can live in `_drafts/` (no date prefix) and are not published.

See the [Chirpy writing guide](https://chirpy.cotes.page/posts/write-a-new-post/) for prompts, Mermaid, math, and
other options.

## Running locally

Requires Ruby 3.x and Bundler.

```sh
bundle install
bash tools/run.sh      # serves at http://127.0.0.1:4000 with live reload
bash tools/test.sh     # production build + html-proofer
```

## Comments

Comments use [giscus](https://giscus.app), backed by this repository's GitHub Discussions (**Announcements**
category). Settings are under `comments.giscus` in [`_config.yml`](_config.yml); moderate them from the Discussions tab.
