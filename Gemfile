source "https://rubygems.org"

# Matches the exact gem versions GitHub Pages builds with, so local previews
# can't drift from production. Do not pin jekyll separately — this gem owns it.
gem "github-pages", group: :jekyll_plugins

gem "webrick", "~> 1.8"   # not bundled with Ruby 3+, needed by `jekyll serve`

group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end
