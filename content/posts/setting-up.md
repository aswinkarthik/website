---
title: "Setting up this Blog"
date: 2019-08-28T17:44:58+05:30
tags: ["blog", "how-to"]
---

I wanted to do a small write up on how I set this blog up. This blog uses [hugo](https://github.com/gohugoio/hugo) to generate content. Things to note about Hugo:

- It is a static site generator.
- Themes are easily pluggable.
- Content can be written as markdown files.

## Steps

### 1. Github Repository

Create a repository to host all your content. This is where you will be maintaining markdown files. So I created https://github.com/aswinkarthik/website.

### 2. Setup using Hugo CLI

Hugo CLI allows us to bootstrap a new site.

```bash
$ hugo new site website
$ cd website && tree
.
|____archetypes
| |____default.md
|____content
|____layouts
|____static
|____config.toml
|____data
|____themes
```

> Note: Please refer Hugo docs for extensive documentation. This blog touches upon only few things I did.

### 3. Choose a theme

Choose a hugo theme from an [exhaustive list](https://themes.gohugo.io/). Add the theme as a submodule to your project. Configure `config.toml` with theme information. Refer [here](https://github.com/aswinkarthik/website/blob/master/config.toml)

```bash
$ git submodule add https://github.com/${THEME_URL}.git themes/${THEME_NAME}
```


### 4. Github Repository to host HTML

We will be writing content only in Markdown files. Once that is done, we need to generate HTML content with appropriate static assets based on our theme. To host this, we need a github repository. This needs to be created in the domain name of the site. We could choose from 2 options

*Use Github's domain*

Create the repository in the name `${USERNAME}.github.io`. Your site will be served with the same URL.

*Use custom domain*

Create a custom domain using some provider. E.g domains.google.com. Create the repository in your name. In my case, I created the repo `aswinkarthik.dev`. Go to the `Settings` page of your repository. In the `GitHub Pages` section, specify your custom domain. Also read [this](https://help.github.com/en/articles/setting-up-an-apex-domain) to point your domain to Github servers.

Whatever domain you choose, make sure to update it in `./config.toml` under `baseURL`. We are at a point where if we put any HTML content into this repository, it will be served as a website. Let's do that.

### 5. Continously deploy with Travis

Now we need to generate HTML content from our markdown files. If we run `hugo` in our codebase, it would generate all those content inside `public` directory. If we could bring that content to our `${USERNAME}.github.io` repository, it will be hosted. Let's automate that part. We are going to use Travis CI's Pages deployer. Create a `.travis.yml` with content


```YAML
sudo: false
dist: trusty
install:
- wget -q "https://github.com/gohugoio/hugo/releases/download/v0.51/hugo_0.51_Linux-64bit.tar.gz"
  -O "hugo.tar.gz"
- tar -xzf hugo.tar.gz
script:
- "./hugo"
deploy:
  provider: pages
  repo: ###${USERNAME}/${REPO}###
  fqdn: ###YOUR DOMAIN###
  target-branch: master
  skip_cleanup: true
  local_dir: public
  github_token: "$GITHUB_TOKEN"
  keep-history: true
  on:
    branch: master
env:
  global:
    secure: ###SECURE_TOKEN###
```

The above steps download hugo and create the public directory by running `./hugo`. The following needs to be configured by you (things that are prefixed with #):

- `deploy.repo`
- `deploy.fqdn`
- `env.global.secure`

To get the secure token, first go your profile `Settings`, then `Developer Settings` and then create a `Personal Access token` with `public_repo` access (it will be under `repo`). Save the token. We need to pass this token to Travis. In order to that, we can encrypt it using Travis CLI and store it in our YAML directly.


```bash
# Install Travis CLI
$ sudo gem install travis

# Login
$ travis login

# Encrypt
$ travis encrypt GITHUB_TOKEN=${THE_TOKEN_YOU_CREATED} -r ${USERNAME}/${YOUR_HUGO_REPO} --add
```

This would have populated the `env.global.secure` section of the `.travis.yml`. You can check-in this now.
Travis will build and publish the HTML files to Github pages. You could access your URL using the domain you chose.

### 6. Future workflow

To create posts

```bash
# This would create a Draft
$ hugo new posts/setting-up.md
```

To view and live reload

```bash
$ hugo server - -D

# In case, if themes are not checked out
$ git submodule update --init
```

To publish, remove this content from your post

```
draft: true
```

Commit & push, it will be automatically deployed!