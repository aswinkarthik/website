---
title: "How I navigate directories?"
date: 2019-08-30T12:45:00+05:30
tags: [ "how-to", "interactive", "cli-hacks"]
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

Now, we have a huge list of results. Pipe the results to [fzf](https://github.com/junegunn/fzf)

```bash
$ fd . -d 5 ${HOME}/project/ -E vendor | fzf
```

![Interactive mode through FZF](/directory-navigation/image-1.png)

The selection is returned as a result of the command. We could choose to handle however we want if its a directory.

```bash
function godir {
    directory=$(fd -E vendor -d 5 . ${HOME}/project/ | fzf --color 16)
    if [ ! -z  "$directory" ]; then
        if [ ! -d "$directory" ]; then
            # Its a file. cd to the directory.
            cd $(dirname "$directory")
        else
            # cd to the directory
            cd $directory
        fi
    fi
}

$ godir # opens an interactive finder now
```

We have a basic interactive finder now. Let's add a little bit of preview feature before deciding to change directory.

> Alternatives: [jhawthorn/fzy](https://github.com/jhawthorn/fzy), [lotabout/skim](https://github.com/lotabout/skim)

### 3. Preview

`fzf` allows us to run a preview command before actually making the selection. Lets do the following,

- If its a directory, list the files.
- If its a file, preview the file

```bash
function previewSelection {
    directory=$1
    if [ ! -z  "$directory" ]; then
        if [ ! -d "$directory" ]; then
            # cat but with syntax highlighting
            bat --color=always "$directory"
        else
            # ls with colors and git-aware
            exa --color=always -l --git --git-ignore -h $directory
        fi
    fi
}

# Export the function
export -f previewSelection
```

Now, in our previous function we change our fzf to preview our selection.

```bash
fd -E vendor -d 5 . ${HOME}/project/ | fzf --color 16 --preview "previewSelection {}"
```

**Preview a directory with exa:**

![preview-a-directory](/directory-navigation/image-2.png)

> Alternatives: You could just use `ls`.

**Preview a file with bat:**

![preview-a-file](/directory-navigation/image-3.png)

> Alternatives: `cat` or any other highlight tool you would want.

### 4. Our final script

```bash
function previewSelection {
    directory=$1
    if [ ! -z  "$directory" ]; then
        if [ ! -d "$directory" ]; then
            # cat but with syntax highlighting
            bat --color=always "$directory"
        else
            # ls with colors and git-aware
            exa --color=always -l --git --git-ignore -h $directory
        fi
    fi
}

# Export the function
export -f previewSelection

function godir {
    directory=$(fd -E vendor -d 5 . ${HOME}/project/ | fzf --color 16 --preview "previewSelection {}" )
    if [ ! -z  "$directory" ]; then
        if [ ! -d "$directory" ]; then
            # Its a file. cd to the directory.
            cd $(dirname "$directory")
        else
            # cd to the directory
            cd $directory
        fi
    fi
}
```

### Tools Reference

- [sharkdp/fd](https://github.com/sharkdp/fd)
- [sharkdp/bat](https://github.com/sharkdp/bat)
- [ogham/exa](https://github.com/ogham/exa)
- [junegunn/fzf](https://github.com/junegunn/fzf)
