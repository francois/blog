source "https://rubygems.org"
ruby RUBY_VERSION

# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima", "~> 2.0"

require "json"
require "open-uri"
versions = JSON.parse(open("https://pages.github.com/versions.json").read)

gem "github-pages", versions["github-pages"]

# If you have any plugins, put them here!
group :jekyll_plugins do
   gem "jekyll-feed", "~> 0.6"
end
