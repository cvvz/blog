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
layout but uses GitHub issue links instead of the legacy Gitalk widget.
No OAuth client secret is required. Discussions are found by the article path
in the issue body, so readers do not need permission to assign issue labels.
The comment key defaults to the first alias, or the current URL for articles
without aliases. New discussion links prefill both the canonical URL and this
stable key. Set `commentPath` only when an older discussion uses a different
URL. The comment links respect `comments: true` in front matter.
