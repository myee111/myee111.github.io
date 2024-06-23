---
layout: post
title:  "How I got Jekyll working with github pages"
date:   2024-06-22 20:34:00 -0700
categories: jekyll
---
## Introduction

1) Install Jekyll as per [jekyll docs, for MacOS](https://jekyllrb.com/docs/installation/macos/).

2) Fork [Chirpy](https://chirpy.cotes.page/posts/getting-started/#option-2-github-fork)

## Fix the `assets` directory

The `assets` directory is a link to Chirpy docs. Delete the `assets` directory and recreate it.

## Configure github actions to deploy the page

Go into `Settings`.

![Github Settings](/assets/lib/githubsettings.png)

Click on `Pages`

![Pages](/assets/lib/githubpages.png)

In the `Source` dropdown box, select `GitHub Actions`.

![Github Actions Config](/assets/lib/githubactionsconfig.png)
