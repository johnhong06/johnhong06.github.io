# johnhong06.github.io

Personal blog of **JH0624** — paper reviews and study notes, written in Korean.

Live site: <https://johnhong06.github.io>

## Structure

| Path | What it holds |
|---|---|
| `_posts/` | Published posts, `YYYY-MM-DD-english-slug.md` |
| `_drafts/` | Unpublished drafts and post templates (not built in production) |
| `_tabs/` | Sidebar pages (About, Categories, Tags, Archives) |
| `assets/img/posts/<slug>/` | Images for each post |
| `depth-anythingv2-test1/` | Videos and raw CSVs for the `실험` post, kept at the URLs of the former project site |
| `_config.yml` | Site settings |

Categories: `논문리뷰` (paper reviews), `공부` (study notes) and `실험` (experiment notes).

## Writing a post

1. Copy a template from `_drafts/` into `_posts/` and rename it to `YYYY-MM-DD-english-slug.md`.
2. Fill in the front matter. Set `math: true` for posts with LaTeX.
3. Push to `main`. GitHub Actions builds and deploys the site.

## Local preview (Docker)

```bash
docker run --rm -it -p 4000:4000 \
  -u "$(id -u):$(id -g)" -e HOME=/tmp -e BUNDLE_PATH=/gems \
  -v "$HOME/.cache/chirpy-bundle:/gems" -v "$PWD:/srv" -w /srv \
  ruby:3.4 bash -c "bundle install && bundle exec jekyll serve -H 0.0.0.0 --drafts"
```

Then open <http://localhost:4000>.

## Credits

- Theme: [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy), set up from [chirpy-starter](https://github.com/cotes2020/chirpy-starter) (v7.6.0). MIT License, see `LICENSE`.
- Post content © JH0624. The MIT License covers the site template only, not the posts.
