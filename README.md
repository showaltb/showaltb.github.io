# showaltb blog

Built with [Jekyll](http://jekyllrb.com), themed with [Lanyon](http://lanyon.getpoole.com), and assembled/reorganized by [Bobby Showalter](http://twitter.com/bobbyshowalter).

## Rakefile

- `rake post["Title"]` creates a new post with the template file (`templates/_post.txt`). The filename is generated with the Jekyll-formatted date and the supplied `["Title"]`. This new post is then automatically opened in the default editor (currently set to `vim`). Note that though the template uses the `.txt` extension, the filetype is changed to `.md` when in use.

- `rake draft["Title"]` is the same as `rake post["Title"]`, but only generates a draft that does not get published.

- `rake publish` displays a numbered list of unpublished drafts. After choosing a draft, the Jekyll-formatted date is automatically added to the filename, and the draft is moved from `_drafts` to `_posts` for publishing. This new post is also listed on the `archive.md` page.

- `rake page["Title"]`, `rake page["Title","path/to/folder"]` creates a new page with the template file (`templates/_page.txt`). New pages are written with Markdown and automatically included in the sidebar (as long as they use `layout: page`).

- `rake build`, `rake build["drafts"]` work the same as `jekyll build` and `jekyll build --drafts`. This command only compiles the site to `_site`; it does not start a server.

- `rake watch`, `rake watch[number]`, `rake watch["drafts"]` build the site (with optional limited `[number]` of posts or with `["drafts"]`), start a server, and automatically rebuild the site when changes are saved.

## Configs

**`_config.yml`**

This file contains all the configuration options for Jekyll. The groups have been arranged alphabetically by header, but can be reordered in whatever way makes sense. Reusable site data should not be placed into this file.

The local server is currently set to use `host: 0.0.0.0` and `port: 3000`; these are customizable and will be used by `rake watch` to serve the site.

**`_data/*.yml`**

These files are for reusable site data. The `key: value` pairs in these files are accessed with liquid output markup: `{{ site.data.<file name>.<key name> }}`. Options for configuring Jekyll itself should be placed in `_config.yml`.

## Styling

### Sass

This project is using the `.scss` version of [Sass](http://sass-lang.com). The main stylesheet with all of the `@import`s is in `public/css/styles.scss`, while the partials are in `public/css/_scss`. Jekyll is already configured to look in this folder for partials, so you can safely omit `public/css/_scss` when using `@import`. Note that all partials' filenames must begin with an underscore (`_partial.scss`).

The Sass output is currently configured as `:compressed`.

### Lanyon theme

The Lanyon theme ships with a few color themes. Unfortunately, the default names are pretty tough to remember. To help use these themes, they have been given useful names in `_data/colors.yml`. To change the theme, open `_layouts/default.html` and locate `<body class="{{ site.data.colors.<color name> }}">` Replace `<color name>` with your desired color to change the theme.

More themes can be added by editing the "Themes" section in `public/css/_scss/_lanyon.scss`.

Lanyon was set to use the PT Sans/Serif typefaces; they have been swapped out with the Source Sans Pro typeface. The original references to PT Sans/Serif have only been commented out in case you would like to switch back.
