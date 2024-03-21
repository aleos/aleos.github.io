---
title: How to post on GitHub Pages?
date: 2024-03-20T00:00:00Z
tags: [swiftui, swift, ios, dev]
draft: true
---

## Use Jekyll

It is a default static site generator for GitHub Pages.
First, install Jekyll on your local machine:

https://mac.install.guide/ruby/13

- install Ruby of the latest version

```zsh
brew install ruby
```

- add the path to the Ruby installation to the PATH environment variable: add this at the end of your `~/.zshrc` or `~/.zprofile` file.

```zsh
if [ -d "/opt/homebrew/opt/ruby/bin" ]; then
  export PATH=/opt/homebrew/opt/ruby/bin:$PATH
  export PATH=`gem environment gemdir`/bin:$PATH
fi
```

and reset the shell session

```zsh
source ~/.zshrc
source ~/.zprofile
```

- install Jekyll

```zsh
gem install jekyll bundler
```
