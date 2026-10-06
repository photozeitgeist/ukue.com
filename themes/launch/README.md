# Launch

A project-site theme for Hugo: a landing-page homepage and clean,
single-column pages. It stands alone, with no other theme needed.

It reads the same content and settings as Ananke sites (posts, pages, tags,
menus, `featured_image`, `media_partners`), so a site can switch without
touching its content. URLs, feeds and the sitemap stay the same.

## Switch a site over

1. Put this folder in the site's `themes/` folder.
2. In `config.toml`, set `theme = "launch"`.
3. Delete the old theme folder (for example `themes/ananke`).

## Optional settings (config.toml)

All optional. Without them the homepage shows the site title and
`params.description` in the header band, then the latest posts.

```toml
[params]
  recent_copy = "Articles"        # heading above the post cards (default "Recent Posts")
  recent_posts_number = 9         # cards on the homepage (default 9)
  brand_color = "#1F3A5F"         # header, hero and footer colour
  accent_color = "#0E7C86"        # buttons and links

[params.hero]                     # homepage header band
  badge = "Short status line"
  badge_url = "/about/"
  title = "Headline"
  subtitle = "One or two sentences."
  image = "/images/screenshot.jpg"
  image_alt = "What the image shows"
  image_link = "/demo/"
  [[params.hero.buttons]]         # first one is the main button
    name = "Main action"
    url = "/demo/"
  [[params.hero.buttons]]
    name = "Second action"
    url = "/about/"

[params.figures]                  # row of key numbers under the hero
  note = "Small print under the row"
  [[params.figures.items]]
    value = "400,000"
    label = "What the number means"

[params.cta]                      # closing band above the footer
  title = "Get in touch"
  text = "One sentence."
  button = "Contact"
  url = "/contact/"
```

Menus:

- `[[menu.main]]` items appear in the top bar and the footer.
- Add `[menu.main.params]` with `button = true` under an item to show it as a
  button at the right end of the top bar.
- `[[menu.footer]]` items (RSS, Sitemap, Archive and so on) appear in the
  footer only.

Other settings it understands: `media_partners` (footer column),
`date_format`, `favicon`, `custom_css` (extra stylesheets from `static/`),
and per page `toc: true` and `show_reading_time: true`.

Posts with `featured_image` show it on their card, not as a page header.
The `form-contact` shortcode works as it did in Ananke.
