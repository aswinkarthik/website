---
title: "How I navigate directories?"
date: 2019-08-30T12:45:00+05:30
draft: true
---

I do not use any `autojump` tools to navigate across my project directories. A simple `bash` function does the job for me. Customization is the main reason.

The tools I use to create the functions can be replaced with GNU tools. I use them because they are fast and have some additional features like respect `.gitignore` file. Let's create the function.

## Steps

### 1. Find directories or files to navigate to 

I need a way to list all the directories I am interested in. This could be directories under your project dir (for me, it is `${HOME}/projects`).

```bash
$ fd . -d 5 ${HOME}/project/ -E vendor
```

This lists all directories in my project directory till 5 depth. By default [fd](https://github.com/sharkdp/fd) ignores git-ignored files and directories. I am also excluding `vendor` directory for my golang projects as sometimes they are not git-ignored. 

> Note: Use any finder instead fd. GNU find will work too.

### 2. Open an interactive fuzzy-finder

Now, we have a hugo list of results. Pipe the results to [fzf](https://github.com/junegunn/fzf)

```bash
$ fd . -d 5 ${HOME}/project/ -E vendor | fzf
```

![Interactive mode through FZF](/directory-navigation/image-1.png)