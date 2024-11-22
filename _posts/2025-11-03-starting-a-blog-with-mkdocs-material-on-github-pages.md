---
title:  "Starting a blog with mkdocs-material on GitHub pages"
date:   2025-11-02 16:06:06 +0200
categories: github markdown blog migration mkdocs material
---

## Get Started with Material for MkDocs

### Installation

[Installation](https://squidfunk.github.io/mkdocs-material/getting-started/)
of Material for MkDocs using `pip` is not recommended in Ubuntu Studio; an
`error: externally-managed-environment` recommends using `apt install` instead:

```
# apt install mkdocs-material mkdocs-material-extensions -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  ghp-import libjs-bootstrap4 libjs-lunr libjs-modernizr libjs-popper.js libjs-sizzle mkdocs
  node-jquery python3-joblib python3-livereload python3-lunr python3-mergedeep python3-nltk
  python3-pathspec python3-platformdirs python3-pymdownx python3-pyyaml-env-tag python3-regex
  python3-tqdm
Suggested packages:
  libjs-es5-shim mkdocs-doc nodejs coffeescript node-less node-uglify python-livereload-doc
  python3-django python3-slimmer python-lunr-doc
Recommended packages:
  prover9
The following NEW packages will be installed:
  ghp-import libjs-bootstrap4 libjs-lunr libjs-modernizr libjs-popper.js libjs-sizzle mkdocs
  mkdocs-material mkdocs-material-extensions node-jquery python3-joblib python3-livereload
  python3-lunr python3-mergedeep python3-nltk python3-pathspec python3-platformdirs
  python3-pymdownx python3-pyyaml-env-tag python3-regex python3-tqdm
0 upgraded, 21 newly installed, 0 to remove and 3 not upgraded.
Need to get 8,721 kB of archives.
After this operation, 41.4 MB of additional disk space will be used.
```

**Note:** Material for MkDocs relies on `watchmedo` from 
[gorakhargosh/watchdog](https://github.com/gorakhargosh/watchdog/tree/master?tab=readme-ov-file#shell-utilities)
so wherever that is installed must be added to the `PATH` variable, e.g. in `.bashrc`

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### Creating a new site

Create a new directory for the new site, and initialize it:

```
$ mkdocs new .
INFO    -  Writing config file: ./mkdocs.yml
INFO    -  Writing initial docs: ./docs/index.md
```

Setup the site name and URL in `mkdocs.yml`

```yml
site_name: My Test Site
site_url: https://this.is.only.a/test
theme:
  name: material
```

Run the local server and follow the link provided to see the new site:

```
$ mkdocs serve
```

Content can be added now to `docs/index.md` and any new files. Changes are picked up
as soon as the files are saved to disk, the site will automatically refresh.

## Create a Blog

[Set up a blog](https://squidfunk.github.io/mkdocs-material/setup/setting-up-a-blog/)
using the built-in `blog` plugin.

Alternatively, create a new repository with the **Use this template** button in the
[Material for Mkdocs Blog Template](https://github.com/mkdocs-material/create-blog?tab=readme-ov-file#a-material-for-mkdocs-blog-template)

## Publish to GitHub Pages

For a quick overview, watch
[Material for MkDocs: Full Tutorial To Build And Deploy Your Docs Portal](https://www.youtube.com/watch?v=xlABhbnNrfI).

Use [GitHub Actions](https://squidfunk.github.io/mkdocs-material/publishing-your-site/#with-github-actions)
to automatically publish the blog on GitHub Pages when pushing commits to the remote repo.

Check [Beautiful Pages on GitLab with mkdocs](https://yodamad.hashnode.dev/beautiful-pages-on-gitlab-with-mkdocs)
for additional inspiration on how to create and publish the blog.
