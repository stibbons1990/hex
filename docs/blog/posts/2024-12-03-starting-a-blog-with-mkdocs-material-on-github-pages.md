---
date: 2024-12-03
categories:
 - github
 - markdown
 - blog
 - migration
 - mkdocs
 - material
title: Starting a blog with mkdocs-material on GitHub pages
---

## Get Started with Material for MkDocs

<!-- more -->

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
$ mkdir notes
$ cd notes
$ mkdocs new .
INFO    -  Writing config file: ./mkdocs.yml
INFO    -  Writing initial docs: ./docs/index.md
```

Setup the site name and URL in `mkdocs.yml`

```yml
site_name: Inadvisably Applied Magic
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
[Material for Mkdocs Blog Template](https://github.com/mkdocs-material/create-blog?tab=readme-ov-file#a-material-for-mkdocs-blog-template).

To create a blog with the built-in `blog` plugin, add it to the
`mkdocs.yml`:

```yaml
site_name: Inadvisably Applied Magic
plugins:
  - blog
theme:
  name: material
```

Now, as soon as the `blog` plugin is loaded, the server crashes and refuses
to start again, because the  `paginate` module cannot be found:

```
$ mkdocs serve
...
  File "/usr/lib/python3/dist-packages/material/plugins/blog/plugin.py", line 38, in <module>
    from paginate import Page as Pagination
ModuleNotFoundError: No module named 'paginate'
```

There is no system package in Ubuntu 24.04 that provides this module,
so it must be installed via `pip`:

```
$ sudo pip3 install paginate
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.
    
    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
    sure you have python3-full installed.
    
    If you wish to install a non-Debian packaged Python application,
    it may be easiest to use pipx install xyz, which will manage a
    virtual environment for you. Make sure you have pipx installed.
    
    See /usr/share/doc/python3.12/README.venv for more information.

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.

$ sudo pip3 install --break-system-packages paginate
Collecting paginate
  Downloading paginate-0.5.7-py2.py3-none-any.whl.metadata (11 kB)
Downloading paginate-0.5.7-py2.py3-none-any.whl (13 kB)
Installing collected packages: paginate
ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
mkdocs-material 9.4.0 requires pymdown-extensions~=10.2, but you have pymdown-extensions 9.5 which is incompatible.
Successfully installed paginate-0.5.7
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv
```

### Add a Post

Posts can be any files under `docs/blog/posts/` so long as they have a `date`
in their metadata section. It is convenient to add the date to the file name,
e.g. `docs/blog/posts/2024-11-01-test.md`, but the `date` in the metadata will
take precedence over that in the file name.

```md
---
date: 2024-11-01
---

# This is only a test

Really, just a test.
```

Other than the `date`, each post can have other
[metadata](https://squidfunk.github.io/mkdocs-material/plugins/blog/#metadata)
define in the metadata block, but the **title** is taken either from the
first top-level heading (`# Title`) or, otherwise, the file name.

Other interesting metadata is the `categories`, to classify and group posts
under the **Archive** section, and the `slug` in case of really long titles
getting URLs too long.

```md
---
date: 2024-11-01
slug: this-is-only-a-test
categories:
  - Search
  - Performance
---

# This is only a test. Honest! Really, just a test, nothing else, promise!

Really, just a test.
```

## Arrange Content

By default, a blog using `mkdocs` built-in plugin will be *under* the main page.

That is, under the main directory (created with `mkdocs new`)
the files are as follows:

    mkdocs.yml              # The configuration file.
    docs/
        index.md            # The homepage.
        ...                 # Other markdown pages and files.
        blog/
            index.md        # The blog's homepage.
                posts/
                    2024-12-01-test.md   # A post

This structure leads to the following URLs:

    example.com/            # The homepage.
    example.com/blog/       # The blog's homepage.
    example.com/blog/2024/11/01/this-is-only-a-test/  # A post

If the main directory is not at the root of a (sub)domain, as is
the case with GitHub Pages, the URLs will more verbose:

    user.github.io/repo/            # The homepage.
    user.github.io/repo/blog/       # The blog's homepage.
    user.github.io/repo/blog/2024/11/01/this-is-only-a-test/  # A post

If the `repo` repository is meant to host **only** a blog,
it would be desirable to use
[the `same-dir` plugin](https://github.com/oprypin/mkdocs-same-dir#mkdocs-same-dir).
This is not used in this site, because there is more than a blog
here; there are also *static pages* such as
[Continuous Monitoring](../../conmon.md).

It can also be good to have a clear separation between the blog,
as an otherwise unorganized timeline of events, and the more
(if not better) organized content in static pages.

### Site-agnostic URLs

To make links to other pages in the same site, and image URLs,
agnostic of where the site is hostead, and whether the blog is
using the `same-dir` plugin or not, `extra` data can be added to
the `mkdocs.yml` configuration and used in content generation via
the `macros` plugin:

```yml
extra:
  site:
    baseurl: http://127.0.0.1:8000/
nav:
  - About This Place: 'index.md'
  - Continuous Monitoring: 'conmon.md'
  - Blog: 'blog/index.md'
plugins:
  - blog:
      pagination: true
      pagination_per_page: 10
  - macros:
      on_undefined: strict
site_name: Inadvisably Applied Magic
```

This may require installing the `macros` pluging in the system;
in Ubuntu this requires installing `mkdocs-macros-plugin`:

```
# sudo apt install mkdocs-macros-plugin
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  python3-termcolor
The following NEW packages will be installed:
  mkdocs-macros-plugin python3-termcolor
0 upgraded, 2 newly installed, 0 to remove and 3 not upgraded.
Need to get 29.1 kB of archives.
After this operation, 124 kB of additional disk space will be used.
```

## Publish to GitHub Pages

To publis the new site to GitHub Pages, replacing the previous
[site based on Jekyll](2023-09-21-starting-a-blog-with-jekyll-on-github-pages.md), first *undo*
some of the settings required by Jekyll, then setup the new flow.

Replace the current flow with the new one

1. Under your repository name, click **Settings**.
   If you cannot see the "Settings" tab, select the  dropdown menu,
   then click **Settings**.
1. In the "Code and automation" section of the sidebar, 
   click **Pages**.
1. Under "Build and deployment", under "Source", select 
   **GitHub Actions**.
1. Follow the link to **create your own**.
1. Name it `ci.yml` and paste the YAML from
   [Publishing your site with GitHub Actions](https://squidfunk.github.io/mkdocs-material/publishing-your-site/#with-github-actions).
1. Go back to "Build and deployment", and under "Source", switch 
   back to **Deploy from a branch** and make sure the selected
   branch is **gh-pages** (and the forder is **/ (root)**).
1. Click on **Commit changes...**, then **Commit changes**.

After this commit the workflow will start and, understandably, fail
with `Error: Config file 'mkdocs.yml' does not exist.` because the
required files are missing. The next commit should include at least

    mkdocs.yml              # The configuration file.
    docs/
        index.md            # The homepage.
        ...                 # Other markdown pages and files.
        blog/
            index.md        # The blog's homepage.
                posts/
                    2024-12-01-test.md   # A post

When trying to use the `macros` plugin, the workflow
will also fail, this time with

```
ERROR   -  Config value 'plugins': The "macros" plugin is not installed
``` 

To fix this, add a step to install the plugin in the
`.github/workflows/ci.yml` just before deploying:

``` yaml
      - run: pip install mkdocs-material 
      - run: pip install mkdocs-macros-plugin
      - run: mkdocs gh-deploy --force
``` 

Now the workflow finishes successfully, and the site
is updated, which means it's mostly empty because the
old content needs to be moved under `docs/blog/posts`.

### Migrate old content

The files under `_posts` will need a number of modifications,
once copied under `docs/blog/posts`:

- metadata `date` must be **only** a date (e.g. `2024-12-03`)
- metadata `categories` must be a list in YAML format
- media files are **moved** under `docs/blog/media/`
  [using the command line](https://docs.github.com/en/repositories/working-with-files/managing-files/moving-a-file-to-a-new-location#moving-a-file-to-a-new-location-using-the-command-line)
  [`git mv`](https://git-scm.com/docs/git-mv).
- media files URLs are then relative to blog posts: `../media/`

```
$ mkdir docs/blog/media
$ git mv assets/media/* docs/blog/media/
$ git ci -m "Move media files to mkdocs blog" .
$ git push
Username for 'https://github.com': stibbons1990
Password for 'https://stibbons1990@github.com': 
...
```

The files under `docs/blog/posts` can also be created by **moving** the
files already existing under `_posts`, then making the necessary
modifications in follow-up commits:

```
$ mkdir docs/blog/posts
$ git mv _posts/*.md docs/blog/posts/
$ git ci -m "Move posts to mkdocs blog" .
$ git push
Username for 'https://github.com': stibbons1990
Password for 'https://stibbons1990@github.com': 

```

This second `git mv` operation will lead to many `WARNING` messages in the
Action run of `mkdocs gh-deploy`, which can be used to track down all stray
references that may need fixing.

**Note:** instead of the password, use a
[fine-grain token for personal access](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token).

## Appendix

For a quick overview, watch
[Material for MkDocs: Full Tutorial To Build And Deploy Your Docs Portal](https://www.youtube.com/watch?v=xlABhbnNrfI).

Check [Beautiful Pages on GitLab with mkdocs](https://yodamad.hashnode.dev/beautiful-pages-on-gitlab-with-mkdocs)
for additional inspiration on how to create and publish the blog.
