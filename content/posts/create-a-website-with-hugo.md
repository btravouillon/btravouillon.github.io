---
title: "Create a Website With Hugo"
date: 2020-11-14T00:16:24-05:00
draft: true
---

Install hugo if requested.

```console
$ curl -L https://github.com/gohugoio/hugo/releases/download/v0.78.2/hugo_0.78.2_Linux-64bit.tar.gz \
    | tar -xz -C ~/.local/bin/ hugo
```

Create the new site.

```console
$ hugo new site btravouillon.github.io
Congratulations! Your new Hugo site is created in /home/bruno/Documents/git/btravouillon.github.io.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
$ git init btravouillon.github.io/
Initialized empty Git repository in /home/bruno/Documents/git/btravouillon.github.io/.git/
$ cd btravouillon.github.io/
```

Add a theme.

```console
$ mkdir themes/geekblog
$ curl -L https://github.com/thegeeklab/hugo-geekblog/releases/download/v0.5.3/hugo-geekblog.tar.gz \
    | tar -xz -C themes/geekblog
```

Edit the config.toml file.

```toml
baseURL = "https://btravouillon.github.io/"
languageCode = "en-us"
title = "Bruno's tech minutes"
theme = "geekblog"

# Required to get well formatted code blocks
pygmentsUseClasses = true
pygmentsCodeFences = true
disablePathToLower = true
enableGitInfo = true
```

Create the first post.

```console
$ hugo new posts/create-a-website-with-hugo.md
```

Edit the newly created file, then commit.

```console
$ git add config.toml archetypes content
$ git commit -m "Initial commit"
```

Start the hugo server.

```console
$ hugo server -D
```

Exit.

Add the GitHub repository as remote.

```console
$ git remote add origin https://btravouillon@github.com/btravouillon/btravouillon.github.io.git
$ git branch -m master main
$ git push -u origin main
```
