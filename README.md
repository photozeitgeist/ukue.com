# Launch theme

Launch is a Hugo theme for project and startup sites. It gives you a landing-page
homepage, clean single-column pages, and post cards instead of a blog list. It reads
the same content and settings as Ananke sites, so a site can switch without touching
its posts.

Everything is set in `config.toml`. Nothing inside the theme folder needs editing.

1. [Install or switch a site](#1-install-or-switch-a-site)
2. [How the homepage is built](#2-how-the-homepage-is-built)
3. [Homepage settings](#3-homepage-settings)
4. [Site settings in config.toml](#4-site-settings-in-configtoml)
5. [Menus](#5-menus)
6. [Front matter for posts and pages](#6-front-matter-for-posts-and-pages)
7. [Other pages](#7-other-pages)
8. [Formatting that comes from your content](#8-formatting-that-comes-from-your-content)

---

## 1. Install or switch a site

1. Put the `launch` folder in the site's `themes/` folder.
2. In `config.toml`, set `theme = "launch"`.
3. Delete the old theme folder, for example `themes/ananke`.

Content, URLs, RSS feeds and the sitemap stay the same.

---

## 2. How the homepage is built

From top to bottom. Only the top bar, the header band, the post cards and the footer
are always there. Every other section is optional: leave its block out of
`config.toml` and the section disappears.

| Section | Set by | Shown when |
|---|---|---|
| Top bar | `title`, `[[menu.main]]` | Always |
| Header band (dark) | `[params.hero]` | Always. With no settings it shows the site `title` and `params.description` |
| Screenshot | `image` in `[params.hero]` | An image is set |
| Key figures | `[params.figures]` | At least one item is set |
| Intro text | `content/_index.md` | That file exists and has text |
| Post cards | `recent_copy`, `recent_posts_number` | The site has posts |
| Contact band | `[params.cta]` | A `title` is set |
| Footer | `title`, `params.description`, menus, `media_partners` | Always |

---

## 3. Homepage settings

### Header band: `[params.hero]`

| Key | What it does | If left out |
|---|---|---|
| `title` | The big headline | Site `title` |
| `subtitle` | One or two sentences under the headline | `params.description` |
| `badge` | Small pill above the headline, for a status line | No pill |
| `badge_url` | Makes the pill a link | Pill is plain text |
| `image` | Screenshot under the band, in a browser-style frame that sits half over the dark band | No screenshot |
| `image_alt` | Describes the screenshot for screen readers and image search | Empty |
| `image_link` | Makes the screenshot a link | Not clickable |
| `[[params.hero.buttons]]` | Buttons under the subtitle, each with `name` and `url`. The first is the solid main button, the others are outlined | No buttons |

```toml
[params.hero]
  badge     = "Alpha: a working engine, heavily tested in simulation"
  badge_url = "/alpha/"
  title     = "A tiny database for sensors"
  subtitle  = "Key-value and time-series on the device, SQL on the gateway, and the same records on both, copied byte for byte."
  image      = "/images/demo-preview.jpg"
  image_alt  = "The AltSql live demo: three simulated devices running the Alpha engine in the browser"
  image_link = "/demo/"

  [[params.hero.buttons]]
    name = "Try the live demo"
    url  = "/demo/"

  [[params.hero.buttons]]
    name = "Explore the Alpha"
    url  = "/alpha/"
```

Tips:

- Keep the badge under about 40 characters if you want it on one line on phones.
- The screenshot shows at up to 1040 px wide. A 1200 × 630 image works well, and the
  same size is right for the social share image (see `images` in section 4).
- The buttons stack full width on phones.

### Key figures: `[params.figures]`

A row of big numbers with a short label under each. Three or four fit best on one
row. On tablets they go two by two, on phones one under another.

| Key | What it does |
|---|---|
| `[[params.figures.items]]` | One figure: `value` is the big text, `label` is the line under it |
| `note` | Small print centred under the row, for caveats such as how the numbers were measured |

```toml
[params.figures]
  note = "Measured on the Alpha code, in simulation. The Beta measures every number again on real chips."

  [[params.figures.items]]
    value = "About 15 KB"
    label = "Sensor build on a Cortex-M4, with key-value, time-series and sync"

  [[params.figures.items]]
    value = "400,000"
    label = "Simulated power cuts in one run, with nothing saved lost"
```

### Post cards

The cards come from the site's main section. Hugo picks it as the section with the
most pages, which is `posts` on these sites, so there's nothing to set.

Each card shows the post's first tag, its title, a short summary and the date. Posts
with a `featured_image` also show that image at the top of the card. The whole card
is clickable.

| Key (under `[params]`) | What it does | Default |
|---|---|---|
| `recent_copy` | Heading above the cards | Recent Posts |
| `recent_posts_number` | How many cards | 9 |

- **Order.** Posts with a `weight` in their front matter come first, lowest number
  first. The rest follow, newest first. AltSql uses `weight: 1`, `2` and `3` to keep
  Introduction, How It Works and The Learned Query Optimizer at the top.
- **Summary.** The post's opening text, cut to about 170 characters. A `description`
  in the front matter replaces it.
- **More posts.** When there are more posts than cards, an "All posts" link appears
  next to the heading and goes to /posts/.

### Contact band: `[params.cta]`

| Key | What it does | If left out |
|---|---|---|
| `title` | Heading of the band | No band at all |
| `text` | Sentence under the heading | No sentence |
| `button` | Button label | Contact |
| `url` | Where the button goes | No button |

```toml
[params.cta]
  title  = "Get in touch"
  text   = "Device makers who would like to try the Beta on their own hardware, and anyone with a question or an industry the project should look at, are welcome to write."
  button = "Contact"
  url    = "/contact/"
```

---

## 4. Site settings in config.toml

### At the top of the file

These go above the first `[section]` line.

| Key | What it does |
|---|---|
| `baseURL` | The live address, such as `"https://altsql.com/"`. Used for full links, the feeds and the sitemap |
| `languageCode` | Page language, such as `"en-us"`. Hugo 0.158 and later print a warning suggesting `locale` instead. It's harmless, so leave it |
| `title` | The site name. It's the text logo in the top bar and footer, the end of every browser-tab title ("The Alpha \| AltSql.com") and the feed title |
| `theme` | `"launch"` |
| `copyright` | Optional. The footer text before the year, as in "© AltSql.com 2026". Without it the site `title` is used. It only works up here: the `copyright` line under `[params]` is ignored |

### `[params]`

| Key | What it does | Default |
|---|---|---|
| `description` | The site's summary. It's the search-result description for the homepage and list pages, the hero subtitle when none is set, and the text under the site name in the footer. Posts and pages get their own description from their opening text | None |
| `author` | Shows "By ..." under every post title. Leave it empty to hide it | Empty |
| `recent_copy` | Heading above the homepage cards | Recent Posts |
| `recent_posts_number` | Number of homepage cards | 9 |
| `brand_color` | The dark colour: top bar, header band, big figures, contact band, footer | `"#1F3A5F"` (navy) |
| `accent_color` | Buttons, links, tag labels, callout borders | `"#0E7C86"` (teal) |
| `images` | Default image for link previews when a page is shared on X, LinkedIn, Slack or WhatsApp, such as `["/images/demo-preview.jpg"]`. Without it shared links show no picture | None |
| `date_format` | How dates look on cards and posts. Write it using Hugo's sample date: `"January 2, 2006"` or `"2 Jan 2006"` | `"January 2, 2006"` |
| `show_reading_time` | `true` adds "3 min read" and the word count under every post title | Off |
| `favicon` | Path to a favicon in `static/`, such as `"/favicon.ico"` | None |
| `custom_css` | Extra stylesheets from `static/`, such as `["/css/extra.css"]` | None |
| `[[params.media_partners]]` | Links in the footer's Media Partners column, each with `name` and `url`. They open in a new tab | None |
| `[params.hero]`, `[params.figures]`, `[params.cta]` | Homepage sections, see section 3 | None |
| `logo`, `copyright` | Left over from Ananke. Not used | |

Colours are hex codes. Hover shades and the light callout background are worked out
from the two colours automatically, so two lines re-colour the whole site:

```toml
[params]
  brand_color  = "#2B2D42"
  accent_color = "#D9480F"
```

### Hugo settings to leave as they are

| Setting | What it does |
|---|---|
| `[permalinks]` `posts` and `pages` = `"/:title"` | Builds each URL from the title. Changing this changes every URL. Editing a post's title later changes its URL too |
| `[taxonomies]` `tag = "tags"` | Makes the tag pages at /tags/ |
| `[pagination]` `pagerSize` | Cards per page on /posts/ and /pages/ |
| `[markup.goldmark.renderer]` `unsafe = true` | Lets posts contain raw HTML. The demo embed and the table wrappers need it |
| `[privacy]` | Switches off Hugo's built-in embeds and trackers: YouTube, X, Instagram, Vimeo, Disqus, Google Analytics |
| `[author]` `name` | Old Hugo setting. Not used |

---

## 5. Menus

```toml
[menu]
  [[menu.main]]
    identifier = "alpha"      # unique name for the item
    name       = "Alpha"      # link text
    url        = "/alpha/"
    weight     = 2            # order, lowest first
  [[menu.main]]
    identifier = "demo"
    name       = "Live Demo"
    url        = "/demo/"
    weight     = 5
    [menu.main.params]
      button = true           # show as a button at the right end of the top bar
  [[menu.footer]]
    identifier = "rss"
    name       = "RSS"
    url        = "/index.xml"
    weight     = 8
```

- **`menu.main`** items appear in the top bar and in the footer's Menu column. The
  link to the page you're on is underlined.
- **`button = true`** moves that item to the right end of the top bar as a button. You
  can have more than one. The `[menu.main.params]` lines must come straight after the
  item they belong to. Button items still appear in the footer's Menu column.
- **`menu.footer`** items appear only in the footer's More column. RSS, Sitemap and
  Archive go here.
- On phones the site name and buttons sit on the first row, with the links on a second
  row.

---

## 6. Front matter for posts and pages

| Field | What it does |
|---|---|
| `title` | Page heading, card title and browser-tab title. It also builds the URL |
| `date` | Shown on the post and its card, and orders posts newest first |
| `draft` | `true` keeps the post off the live site |
| `tags` | The first tag is the small label above the title on the post and on its card. All tags appear as chips at the end of the post, linking to their tag pages |
| `featured_image` | Image at the top of the post's card. It isn't shown as a page header, so keep the image in the post body as well |
| `weight` | Optional. Pins posts to the top of the homepage and lists, lowest number first |
| `type: page` | For pages such as About or Contact: title and text only, no date, tags or related posts |
| `url` | Optional. A fixed address, such as `"/alpha/"` |
| `description` | Optional. Replaces the automatic search-result description and card summary |
| `toc: true` | Optional. Adds a Contents box at the top, built from the headings |
| `show_reading_time: true` | Optional. Reading time for this post only |
| `author` | Optional. "By ..." for this post only |
| `images` | Optional. Share image for this post only, such as `["/images/cold-chain.png"]` |

Related posts at the bottom of each post are picked by Hugo from shared tags and
nearby dates.

---

## 7. Other pages

- **Posts:** a light header band with the first tag, the title and the date, then the
  text, the tag chips and related posts.
- **Pages** (`type: page`): the title and the text.
- **/posts/ and /pages/:** cards, split into pages by `pagerSize`.
- **/tags/:** every tag with its number of posts and their titles.
- **A tag page, such as /tags/case-studies/:** cards for the posts with that tag.
- **/archive/:** all posts grouped by year. It needs `content/archive/_index.md` with
  `title: "Archive"`, as on AltSql. It lists the `posts` section only.
- **404:** "This is not the page you were looking for", with a button to the homepage.

---

## 8. Formatting that comes from your content

No settings needed. The theme styles these on its own:

- **Callouts.** A blockquote, such as `> **Where it stands.** Some text`, becomes a
  tinted box with a coloured left edge.
- **Captions.** A line in italics straight after an image becomes a centred caption.
  A line in italics straight after a table becomes a small note.
- **Intro paragraph.** The first paragraph of a post or page is set a little larger.
- **Wide tables.** Wrap a wide table in `<div style="overflow-x:auto">` and `</div>`,
  with a blank line before and after the table, and it scrolls sideways on phones
  instead of squeezing.
- **Embedded apps.** An `<iframe>` on its own line whose `src` starts with `/` (a page
  on the same site, like the live demo) gets up to 1200 px of width instead of the
  740 px text column.
- **Contact form.** The `form-contact` shortcode works as it did in Ananke:
  `{{< form-contact action="https://your-form-service/endpoint" >}}`
