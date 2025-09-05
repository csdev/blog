---
title: "Migrating My Blog to Docusaurus and Cloudflare Workers"
description: >-
  A walkthrough of how I migrated my blog from Jekyll on GitHub Pages
  to Docusaurus on Cloudflare Workers.
tags:
  - other
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

When I initially published my developer blog, I chose to use Jekyll and
GitHub Pages. The decision was simple. I already used GitHub for repository
hosting, so I had easy access to GitHub Pages and its built-in support for
Jekyll. With Jekyll, I could generate a static website from Markdown posts,
allowing me to focus on my blog content instead of wrangling with frontend
code and infrastructure. The framework was (and still is) popular and
well-supported, so there were plenty of tutorials available.

However, GitHub Pages has some major shortcomings. It has no support
for pre-production environments, so you are forced to release directly
to prod with no validation in between. Although my personal blog is not a
mission-critical service, as someone with an SRE background, this situation
always made me nervous.

<!-- truncate -->

Worsening that fear was the difficulty of getting GitHub Pages builds
to be fully reproducible. The GitHub Actions workflow, Gemfile, and
local Ruby environment all had to have matching, compatible package
versions. Version pins had to be carefully kept in sync across multiple
configuration files:

<Tabs>
  <TabItem value="gha" label="GitHub Actions" default>
```yml
jobs:
  build:
    steps:
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          working-directory: ./site
          ruby-version: '2.7.4'
          rubygems: '3.4.22'
          bundler-cache: true
          cache-version: 1
```
  </TabItem>

  <TabItem value="dockerfile" label="Dockerfile">
```Dockerfile
FROM ruby:2.7.4-slim-bullseye

RUN gem update --system '3.4.22'  # this also provides bundler 2.4.22
```
  </TabItem>

  <TabItem value="gemfile" label="Gemfile">
```ruby
source "https://rubygems.org"

gem "ffi", "~> 1.16.3"  # ruby 2.7 needs a specific version of ffi

gem "jekyll", "~> 3.9.5"
gem "minima", "~> 2.5.1"
gem "github-pages", "~> 231", group: :jekyll_plugins
```
  </TabItem>
</Tabs>

Finally, Jekyll was another source of frustration. The default themes were
missing basic features like URL fragments for post headings, line numbering
in code blocks (with the `GFM` parser), and a dark mode toggle. Implementing
them would require custom themes and various configuration hacks.

## Introducing Docusaurus

[Docusaurus](https://docusaurus.io/) is a static site generator designed for
publishing project documentation and blogs. It is based on React and
[MDX](https://mdxjs.com/), which means you can write Markdown posts with
embedded components. The tabbed code blocks I used above is one example:

````js
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs>
  <TabItem value="gha" label="GitHub Actions" default>
```yml
jobs:
  build:
    steps:
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          working-directory: ./site
          ruby-version: '2.7.4'
          rubygems: '3.4.22'
          bundler-cache: true
          cache-version: 1
```
  </TabItem>
</Tabs>
````

The Docusaurus default theme is clean and modern. Navigation elements,
styles, and SEO elements are all customizable via `docusaurus.config.js`.
There are official integrations with numerous hosting providers, and
plugins for extended functionality.

## The Migration Process

### Initial Scaffolding

The first step was to set up my Node development environment and initialize
a new Docusaurus project:

```sh
nvm install 22
nvm use 22

npx create-docusaurus@latest blog classic

cd blog
git init
```

The new project contains some sample documentation and blog posts which
we can already build and serve locally:

```sh
npx docusaurus start
```

The development server runs on `localhost:3000` and automatically reloads
on code changes.

### Configuration

Now it was time to begin configuring and customizing. I started by adding
my website info at the top of the `config` object in `docusaurus.config.js`:

```js title="docusaurus.config.js"
/** @type {import('@docusaurus/types').Config} */
const config = {
  title: 'code soup',
  tagline: 'random thoughts on software development',
  url: 'https://csang.dev',
  baseUrl: '/',
}
```

Since I am not hosting any documentation on my website, I enabled
_blog-only mode_ by setting `docs: false` and changing the base path of the
blog to the root (`/`):

```js title="docusaurus.config.js"
/** @type {import('@docusaurus/types').Config} */
const config = {
  presets: [
    [
      'classic',
      /** @type {import('@docusaurus/preset-classic').Options} */
      ({
        docs: false,
        blog: {
          routeBasePath: '/',
          blogTitle: 'code soup',
          blogDescription: 'random thoughts on software development',
        },
      }),
    ],
  ],
}
```

I also had to set the website title and description again, so they would
appear properly in `<meta>` tags on all blog pages.

### Migrating Files

Next, I deleted the Docusaurus sample pages and copied over my own blog posts,
static assets, and about page from the old Jekyll repository.

```sh
mv $OLD_BLOG/site/_posts/* blog/
mv $OLD_BLOG/site/assets/img/* static/img/
mv $OLD_BLOG/site/about.md src/pages/
```

Relative URLs in each post had to be updated based on the new directory
structure.

### Fixing Compile Errors

#### Templating

A few of the migrated pages failed to compile since they were using
Jekyll-specific syntax. For example, one blog post had a code example
containing `{{ }}` blocks, which are normally reserved for template
substitution in Jekyll and needed to be escaped with `{% raw %}`:

````markdown
{% raw %}
```sh
docker images --filter 'reference=golang' \
    --format '{{ .Repository }}:{{ .Tag }}\t{{ .Size }}'
```
{% endraw %}
````

In Docusaurus, `{{ }}` within a code block has no special meaning,
and the `{% raw %}` escape is invalid JSX, so this code block can
be written normally without it:

````markdown
```sh
docker images --filter 'reference=golang' \
    --format '{{ .Repository }}:{{ .Tag }}\t{{ .Size }}'
```
````

#### Inline Styles

Another error occurred with a license image on my about page that
contained an inline style:

```html
<img alt="Creative Commons License" style="border-width:0"
    src="https://i.creativecommons.org/l/by/4.0/88x31.png" />
```

In Docusaurus, inline styles must use JSX style objects instead of raw
CSS strings:

```html
<img alt="Creative Commons License" style={{ borderWidth: 0 }}
    src="https://i.creativecommons.org/l/by/4.0/88x31.png" />
```

### Page Attributes

There are several Jekyll-specific attributes for setting the layout,
title, and category of each blog post:

```yml
---
layout: post
title: "An Intro to Dev Containers in VS Code"
category: docker
---
```

Converting these to the corresponding Docusaurus attributes was straightforward.
The `category` is replaced with `tags`, which are defined in a `tags.yml`
file. Docusaurus also supports a `description` field for SEO.

```yml
---
title: "An Intro to Dev Containers in VS Code"
description: >-
  Using Dev Containers in Visual Studio Code to provide reproducible
  local development environments.
tags:
  - docker
---
```

Lastly, I had to add truncate markers to each page. By default, Docusaurus
shows a preview of the first 10 blog posts on the home page. Everything
above the `<!-- truncate -->` marker is included in the preview. Without
the marker, the entire post is displayed, which can be an issue for larger
articles.

## Deploying to Production

I decided to use Cloudflare for the free pricing tier, support for preview
builds, and integrations with related infrastructure such as DNS hosting.
To ensure a zero-downtime migration, first I deployed my site to a
`workers.dev` domain where I could perform testing.
Since Jekyll blogs use a different URL format than Docusaurus, I also
had to configure some server-side redirects to avoid breaking bookmarks
for existing users.

```txt title="_redirects"
/assets/img/:filename /img/:filename 301
/topics /tags 301

/docker/2023/01/18/docker-devcontainers.html /2023/01/18/docker-devcontainers 301
/python/2023/01/20/python-annotations.html /2023/01/20/python-annotations 301
/docker/2023/01/24/docker-alpine.html /2023/01/24/docker-alpine 301
/security/2023/04/03/security-oauth2-tokens.html /2023/04/03/security-oauth2-tokens 301
```

Once everything looked good, I switched the production domain
`csang.dev` from the old GitHub Pages deployment to the new Cloudflare one.
Synthetics monitoring on my site confirmed that the cutover went smoothly.
Now I am ready to archive my old repository and begin publishing new content
for my updated site.
