---
title: "Making Hugo Work in Github Pages With a Path"
date: 2023-03-10T12:43:07-05:00
draft: false
---

## Problem

The publish step with a base path does not do the right thing when the site is published to GitHub pages.

```shell
# this does not do the right thing with and without the base path set to the project name in config.toml
hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
```

## Solution

1. Set the base URL to `/` in [config.toml](https://github.com/stand-sure/hugo-pages-site/blob/main/config.toml) in this repository.
2. Remove the `--baseURL` argument in the workflow step. See [.github/workflows/hugo.yml](https://github.com/stand-sure/hugo-pages-site/blob/main/.github/workflows/hugo.yml) in this repository.
