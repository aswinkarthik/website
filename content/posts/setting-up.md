---
title: "Setting up this Blog"
date: 2019-08-28T17:44:58+05:30
draft: true
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

Choose a hugo theme from an [exhaustive list](https://themes.gohugo.io/). Add the theme as a submodule to your project.

```bash
$ git submodule add https://github.com/${THEME_URL}.git themes/${THEME_NAME}
```

Configure `config.toml` with theme information. Refer [here](https://github.com/aswinkarthik/website/blob/master/config.toml)

### 4. Github Repository to host HTML

We will be writing content only in Markdown files. Once that is done, we need to generate HTML content with appropriate static assets based on our theme. To host this, we need a github repository. 

