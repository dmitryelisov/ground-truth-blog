# Ground Truth

The engineering publication by Teramind Engineering, served by GitHub Pages at
https://dmitryelisov.github.io/ground-truth-blog/.

- `_posts/YYYY-MM-DD-slug.md` — published articles. Front matter carries `title`, `description`,
  `author`, `pillar` and an optional `faq` list (`q` / `a`), which renders both the visible FAQ and
  the `FAQPage` JSON-LD.
- `_drafts/` — work in progress. Jekyll does not publish it.
- `llms.txt` — generated from the post list.

Built by GitHub Pages' own Jekyll (theme `minima`, plugins `jekyll-feed`, `jekyll-seo-tag`,
`jekyll-sitemap`). There is no Gemfile and no Actions workflow.
