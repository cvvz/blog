# How-to

```shell
# Download Hugo v0.76.5
wget https://github.com/gohugoio/hugo/releases/download/v0.76.5/hugo_0.76.5_Linux-64bit.tar.gz
tar zxvf hugo_0.76.5_Linux-64bit.tar.gz
sudo mv hugo /usr/local/bin/

# Prepare the resource
git clone https://github.com/cvvz/blog.git && cd blog
mkdir themes && cd themes && git clone https://github.com/cvvz/hermit.git 
cd .. && git clone https://github.com/cvvz/cvvz.github.io.git public

# Test locally
hugo server -D

# Deploy to the Internet
./deploy.sh
```

## Articles and legacy URLs

All articles, including reading notes, live in `content/post/`. Keep each
article's original `date`; the theme groups the list by year, newest first.
The section title is configured in `content/post/_index.md`.

Migrated reading notes declare their old `/read/<slug>/` URLs in `aliases`.
Hugo generates HTML redirects to their new `/post/<slug>/` URLs, and `/read/`
redirects to `/post/`. The RSS-only `content/read/_index.md` and
`layouts/read/rss.xml` keep `/read/index.xml` serving the unified article feed
for existing subscribers.

The project-level `layouts/post/single.html` preserves the theme's article
layout and embeds Utterances through `layouts/partials/comments.html` for
articles with `comments: true`. Readers sign in with GitHub to comment without
leaving the article's comment UI. No OAuth client secret belongs in the site.

## Comments

Install the [Utterances GitHub App](https://github.com/apps/utterances) with
access only to the public `cvvz/cvvz.github.io` repository. Issues must remain
enabled. The app needs issue read/write access, not code write access.

`data/comment_issues.json` maps existing Gitalk discussion keys to their issue
numbers, so Utterances displays the same discussions instead of creating
duplicates. If an old key has multiple threads, retain the one Gitalk selected
(the newest matching issue). Do not delete or recreate existing issues.

The stable key defaults to the first alias, or the current URL for articles
without aliases. Set `commentPath` only when an older discussion uses a
different URL. For new articles without a mapped issue, Utterances uses this
key as `issue-term` and creates a discussion when the first comment is posted.
Readers do not need permission to assign issue labels.

`static/utterances.json` becomes `utterances.json` at the publication repository
root and lists the production origins permitted by the widget. Update it if
the site domain changes. Local previews can display public comments, but are
not listed as allowed posting origins. The widget uses the light theme to
match the blog and provides a GitHub fallback link if it cannot load.
