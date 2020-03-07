# Website

Source code that powers my website

[![Build Status](https://travis-ci.org/aswinkarthik/website.svg?branch=master)](https://travis-ci.org/aswinkarthik/website)

## To Run locally

```bash
## For dev mode
$ hugo server -D

## To generate ./public directory
$ hugo
```

To update theme submodules

```bash
$ git submodule update
$ git submodule update --remote themes/erblog
```