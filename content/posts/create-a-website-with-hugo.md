---
title: "Create a Website With Hugo"
date: 2020-11-21T17:34:24-05:00
tags: ["hugo"]
draft: false
---

![Hugo Logo](/img/hugo-logo.png) + ![GitHub Pages Logo](/img/gh-pages.png)

[Hugo][1] is a static website generator written in Go. This is a framework that
helps to quickly and efficiently build a website from templates, themes and
markdown content.

In this blog post, I will demonstrate how to build a static website with
[Hugo][1] and publish it to [GitHub Pages][2]. This post does not cover the
initialization of the GitHub Pages which is [documented here][3].

Initial Setup of the Website
----------------------------

There are several ways to install Hugo, which depend on your operating system.
This is well explained in the [Hugo documentation][4].

For Linux users, the fastest way to get started with Hugo is to download and
install the portable binary somewhere in your PATH:

```console
$ curl -L https://github.com/gohugoio/hugo/releases/download/v0.78.2/hugo_0.78.2_Linux-64bit.tar.gz \
    | tar -xz -C ~/.local/bin/ hugo
```

Once the hugo binary is installed, you can create the new site. If you want to
publish your website with GitHub Pages, create the project site
`username.github.io` (where "username" is your actual GitHub user name or
organization):

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
```

Initialize a git repository in the newly created site directory:

```console
$ git init btravouillon.github.io/
Initialized empty Git repository in /home/bruno/Documents/git/btravouillon.github.io/.git/
```

The next step to setup the Hugo website is to add a theme. It took me some time
to find a theme that would fit my needs, but I finally came across the
[geekblog theme][5] which is very nice to render IT related blog posts with a
lot of code/console snippets.

```console
$ cd btravouillon.github.io/
$ mkdir themes/geekblog
$ curl -L https://github.com/thegeeklab/hugo-geekblog/releases/download/v0.5.3/hugo-geekblog.tar.gz \
    | tar -xz -C themes/geekblog
```

Once the theme is installed, you must edit the configuration file to enable it.
The snippet below is a minimal `config.toml` to get a proper rendering of the
website:

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

There is an exhaustive configuration file available [here][6] for this theme.

To ensure the configuration is correct, start the built-in live server on your
system. Hugo will process the configuration, templates and theme to build the
static website:

```console
$ hugo server -D
Start building sites …
[...]
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

Open your browser and navigate to the website to check the build is correct.
Once you are done, you can stop the server with Ctrl+C.

Add content to the Website
--------------------------

In the previous step, Hugo built a static website with no content. Now is the
right time to add content to our website. For this, create the first post:

```console
$ hugo new posts/create-a-website-with-hugo.md
```

Edit the newly created file with the content you want to share. Once the
content is ready to be published, add the files to the git index and commit the
changes:

```console
$ git add config.toml archetypes content
$ git commit -m "Initial commit"
```

{{< hint info >}}
**NOTE**\
I don't add the `themes` directory to the git index on purpose, I don't want to track the theme in this git project.
{{< /hint >}}

Add the GitHub repository as remote.

```console
$ git remote add origin https://btravouillon@github.com/btravouillon/btravouillon.github.io.git
$ git branch -m master main
$ git push -u origin main
```

The source files of the website are now managed with git in the **main** branch.

Build and publish the Website
-----------------------------

The next step is to build the static website with Hugo and push the output to a
dedicated branch in our git project. The convention is to name this branch
**gh-pages**. The instructions below are copied from the [Hugo documentation][7].

Add the `public` folder to the `.gitignore` file:

```console
$ echo "public" >> .gitignore
```

Initialize the `gh-pages` branch:

```console
git checkout --orphan gh-pages
git reset --hard
git commit --allow-empty -m "Initializing gh-pages branch"
git push origin gh-pages
git checkout main
```

Now check out the gh-pages branch into your public folder using git’s worktree feature:

```console
$ git worktree add -B gh-pages public origin/gh-pages
```

Build the website:

```console
$ hugo
Start building sites …

                   | EN
-------------------+-----
  Pages            |  7
  Paginator pages  |  0
  Non-page files   |  0
  Static files     | 41
  Processed images |  0
  Aliases          |  3
  Sitemaps         |  1
  Cleaned          |  0

Total in 123 ms
```

Commit and push the website:

```console
$ cd public/
$ git add --all
$ git commit -m "Publishing to gh-pages"
$ git push origin gh-pages
```

In your GitHub project, go to **Settings > GitHub Pages** and update the
**Source** to use the branch `gh-pages` and click the **Save** button. Within a
couple of minutes, GitHub Pages will update the content of the website online
with the content of branch **gh-pages**.

If you want to automate the build process, read my next blog post [Automate
Hugo builds with GitHub Actions][8].

[1]: https://gohugo.io/
[2]: https://pages.github.com
[3]: https://docs.github.com/en/free-pro-team@latest/github/working-with-github-pages/getting-started-with-github-pages
[4]: https://gohugo.io/getting-started/installing/
[5]: https://themes.gohugo.io/hugo-geekblog/
[6]: https://hugo-geekblog.geekdocs.de/posts/getting-started/#configuration
[7]: https://gohugo.io/hosting-and-deployment/hosting-on-github/#preparations-for-gh-pages-branch
[8]: {{< relref "automate-hugo-builds-with-github-actions.md" >}}
