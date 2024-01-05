---
layout:     post
title:      "Generating redirects to old posts in Jekyll"
date:       2024-01-05
comments:   true
tags:       meta jekyll
---

# Problem 

Today I looked at my blog analytics and noticed that somehow I still get requests for very old blog posts, but using a different URL format.  These return a `404`error.

The old blog used `octopress`[^octopress] and apparently used permalinks like `blog/year/month/date/title`.

# Redirect Plugin

So how can we cause Jekyll to recognise these URLs and redirect automatically to the correct post?

Some googling revealed that there's a [Jekyll plugin](https://github.com/jekyll/jekyll-redirect-from) for that.

# Configuration

I just did as the plugin documentation told:

1. Added `gem 'jekyll-redirect-from'` to the `Gemfile`
2. Rebuilt the bundle `bundle`

# Usage

So for every post I want to have Jekyll redirect **to**, I need to add list of
paths to redirect **from** in the posts's YAML front-matter.

For example, this:

```markdown
---
layout: post
title: "Jython, Oracle and JDBC"
date: 2012-02-28 08:21
comments: true
tags: jython python howto
redirect_from:
  - /blog/2012/02/28/jython-oracle-jdbc
---

I recently had the need to access a new table in an existing oracle database
...
```

Redirects form `https://seletz.github.io/blog/2012/02/28/jython-oracle-jdbc` to
whatever URI Jekyll generates for that post.  Nice!

---

[^octopress]:  [octopress](http://octopress.org) is based on Jekyll but it's
    more involved and I couldn't get it back to working, so I reverted to plain
    Jekyll.
