---
title: "Automate Hugo builds with GitHub Actions"
date: 2020-11-29T18:28:33-05:00
tags: ["hugo", "automation"]
draft: false
---

The sources of this blog are tracked in a git repository hosted on GitHub. The
editable content is kept in the default branch `main`, while the static pages
built by Hugo are stored in the `gh-pages` branch. The GitHub Pages hosting
service is configured to use the content of `gh-pages` as a source.

The process to build and publish a website with Hugo, [explained in a previous
blog post][1], requires several manual steps. In a world of automation, I
strongly believe that if you have to do it twice, you should automate it.

In this blog post, I will leverage the power of [GitHub Actions][2] to create a
workflow that will automate the build and publication of this website.

![GitHub Actions Logo](/img/gh-actions.png)

Define the workflow
-------------------

At the time of this writing, the complete workflow to publish new content to
the website is:

1. Checkout the content locally
2. Add or edit some content
3. Commit and push the changes to the `main` branch 
4. Download the theme
5. Add a worktree to the `gh-pages` branch
6. Build the website
7. Commit the changes to the `gh-pages` branch
8. Push to GitHub

I want to automate steps 4 to 8 each time a change is pushed to the `main`
branch (step 3).

The actions defined in the automated workflow will run on GitHub hosted
runners. Any time an action starts, a new runner will be allocated to run the
workflow. Each runner provides a clean instance, so the workflow must setup the
environment for each build.

The automated workflow steps will be:

1. Checkout the `main` branch
2. Install Hugo
3. Download the theme
4. Checkout the `gh-pages` branch in the public directory
5. Build the website
6. Commit and push the changes to the `gh-pages` branch

Create the workflow template
----------------------------

The GitHub Actions workflows are YAML files stored in the `.github/workflows`
directory of the repository. A basic workflow template must include a name, a
list of events that trigger the workflow and a list of jobs.

The automation must trigger the GitHub Actions workflow for any push to the
`main` branch. I define a single job named *build* that will run in an Ubuntu
20.04 VM.

```yaml
name: Build the Hugo website
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-20.04
    steps:
      [...list of steps...]
```

Checkout the content
--------------------

The first step of the *build* job is to checkout the content of the `main`
branch in the working directory. 

GitHub provides an [action to checkout the repository][3], which greatly
simplify the process.

Add the step below to the workflow file:

```yaml
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0
```

This will checkout the branch that triggered the workflow.

Only a single commit is fetched by default. Because I want to update the last
modification date for each page using the last git commit date for that content
file (`enableGitInfo = true` in *config.toml*), this action must fetch all
history for the branch with `fetch-depth: 0`.

Install Hugo
------------

The brew command is installed in the Ubuntu runners, thus it is possible to
[install Hugo with Homebrew][4].

```yaml
    steps:
      [...]
      - name: Install Hugo
        run: brew install hugo
```

Download the theme
------------------

This blog uses the [geekblog theme][5]. To avoid any surprise with the static
pages generated by Hugo, I prefer to stick with a known release of the theme
for automation, defined with the environment variable `GEEKBLOG_RELEASE`.  Any
new release of the theme must be tested locally first, before updating the
value of `GEEKBLOG_RELEASE`.

Edit the workflow to set the release of the theme then add
a new step to download the release from the project page:

```yaml
    env:
      GEEKBLOG_RELEASE: 'v0.7.0'
    steps:
      [...]
      - name: Download the geekblog theme
        run: |
          curl -L -O https://github.com/thegeeklab/hugo-geekblog/releases/download/${GEEKBLOG_RELEASE}/hugo-geekblog.tar.gz
          mkdir -p themes/geekblog
          tar -xzf hugo-geekblog.tar.gz -C themes/geekblog
          rm -f hugo-geekblog.tar.gz
```

Checkout the gh-pages branch in the public directory
----------------------------------------------------

The source branch of my GitHub Pages is `gh-pages`. Since Hugo generates the
static website in the *public* directory, add a step to checkout the branch
`gh-pages` to the *public* directory. Let's use the checkout action with
additional parameters:

```yaml
    steps:
      [...]
      - name: Checkout the gh-pages branch in the public directory
        uses: actions/checkout@v2
        with:
          ref: gh-pages
          path: public
```

Build the website
-----------------

Cleanup the content of the *public* directory before the build:

```yaml
    steps:
      [...]
      - name: Cleanup the public directory
        run: rm -rf public/*
```

The build with Hugo is straightforward. Add a step to run the hugo command in
the workflow:

```yaml
    steps:
      [...]
      - name: Build the website
        run: hugo
```

Commit and push to the gh-pages branch
--------------------------------------

Once the website is generated, the last steps are to commit and push the
changes to the `gh-pages` branch for GitHub Pages to update the website.

There is an action available in the GitHub Marketplace that can handle both
steps at once: [git-auto-commit][6].

Add the step below to the workflow file:

```yaml
    steps:
      [...]
      - name: Commit and push to the gh-pages branch
        uses: stefanzweifel/git-auto-commit-action@v4
        with:
          commit_message: "actions: publish to gh-pages"
          branch: gh-pages
          repository: public/
```

Enable the workflow in GitHub Actions
-------------------------------------

The workflow file is now ready. The aggregated workflow file is available
[here][7] for review. Commit it to the `main` branch and push it to the GitHub
repository.

```console
$ git checkout main
$ git add .github/workflows/build-the-hugo-website.yml
$ git commit -m "Add GH Actions workflow to automate Hugo builds"
$ git push
```

The push will immediately trigger an action with this workflow because the
command pushes to the `main` branch. Next updates to this branch, be it a new
blog post, a new page or even a configuration change will trigger this workflow
and update the website automatically.

From a user perspective, the new workflow to publish content to the website is:

1. Checkout the `main` branch locally
2. Add or edit content
3. Commit and push

This is definitely simpler. :smiley:

[1]: {{< relref "create-a-website-with-hugo.md" >}}
[2]: https://github.com/features/actions
[3]: https://github.com/marketplace/actions/checkout
[4]: https://gohugo.io/getting-started/installing/#homebrew-linux
[5]: https://themes.gohugo.io/hugo-geekblog/
[6]: https://github.com/marketplace/actions/git-auto-commit
[7]: https://github.com/btravouillon/btravouillon.github.io/blob/a911c6e181e583838b195271d7b3cb1644633332/.github/workflows/build-the-hugo-website.yml