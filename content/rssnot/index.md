---
layout: layouts/base.njk
eleventyNavigation:
  key: RSS
  order: 4
---
{%- css %}{% include "node_modules/prismjs/themes/prism-okaidia.css" %}{% endcss %}
{%- css %}{% include "public/css/prism-diff.css" %}{%- endcss %}
# RSS

I was fooling around with the blog, trying to put a link to the feed in the nav-bar by adding
front matter to the `feed.njk` file.
But no link was forthcoming :/

The trouble turned out to be (`feed.11tydata.js`):

```json
module.exports = {
  eleventyExcludeFromCollections: true
}
```

Which make sense, but navigation is fed like:

```diff-njk
for entry in collections.all | eleventyNavigation 
...
endfor
+<li class="nav-item"><a href="/feed/feed.xml">RSS</a></li>
```

which means adding front matter to `feed.njk` is a bust.

You can see where I hacked in the RSS link, which did work!

Then I noticed:

```html
<link rel="alternate" href="/feed/feed.xml" type="application/atom+xml" title="{{ metadata.title }}">
<link rel="alternate" href="/feed/feed.json" type="application/json" title="{{ metadata.title }}">
```

Soo..

Ask for `application/atom+xml` or `application/json` from `{{ metadata.url }}` and boom!

And I got to mess with the nav-bar via front matter with this page : )

