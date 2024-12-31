---
date: 2024-12-03
categories:
 - github
 - markdown
 - blog
 - migration
 - mkdocs
 - material
---

# Starting a blog with mkdocs-material on GitHub pages

After about a year since
[starting a blog with Jekyll on GitHub pages](2023-09-21-starting-a-blog-with-jekyll-on-github-pages.md),
the time has come to step this blog's game up. Jekyll is good,
but it makes copying text a bit hard sometimes (hard to see)
and code blocks are lacking some features like showing file
names and highlighting specific lines. These features, and more,
are available in
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/),
another Markdown-based documentation framwork that looks really good!

## Get Started with Material for MkDocs

For a tutorial, see [Material for MkDocs: Full Tutorial To Build And Deploy Your Docs Portal](https://youtu.be/xlABhbnNrfI?si=b17fSmSDjA6yzpeq).

![type:video](https://www.youtube.com/embed/xlABhbnNrfI)

For more inspiration, see also
[Beautiful Pages on GitLab with mkdocs](https://yodamad.hashnode.dev/beautiful-pages-on-gitlab-with-mkdocs).

<!-- more -->

### Installation

[Installation](https://squidfunk.github.io/mkdocs-material/getting-started/)
of Material for MkDocs using `pip` but this is not recommended in Ubuntu Studio:

``` console
$ pip install mkdocs-material
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.
```

Install using `apt install` instead:

??? terminal "`# apt install mkdocs-material mkdocs-material-extensions -y`"

    ``` console
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

!!! note

    Material for MkDocs relies on `watchmedo` from 
    [gorakhargosh/watchdog](https://github.com/gorakhargosh/watchdog/tree/master?tab=readme-ov-file#shell-utilities)
    so wherever that is installed must be added to the `PATH` variable, e.g. in `.bashrc`
    ```bash
    export PATH="$HOME/.local/bin:$PATH"
    ```

### Creating a new site

Create a new directory for the new site, and initialize it:

``` console
$ mkdocs new .
INFO    -  Writing config file: ./mkdocs.yml
INFO    -  Writing initial docs: ./docs/index.md
```

Setup the site name and URL in `mkdocs.yml`

``` yaml title="mkdocs.yml"
site_name: Inadvisably Applied Magic
theme:
  name: material
```

Run the local server and follow the link provided to see the new site:

??? terminal "`$ mkdocs serve`"

    ``` console
    $ mkdocs serve
    ...
    INFO    -  Building documentation...
    INFO    -  [macros] - Macros arguments: {'module_name': 'main',
              'modules': [], 'render_by_default': True, 'include_dir': '',
              'include_yaml': [], 'j2_block_start_string': '',
              'j2_block_end_string': '', 'j2_variable_start_string': '',
              'j2_variable_end_string': '', 'on_undefined': 'strict',
              'on_error_fail': False, 'verbose': False}
    INFO    -  [macros] - Extra filters (module): ['pretty']
    INFO    -  Cleaning site directory
    INFO    -  Documentation built in 6.55 seconds
    INFO    -  [21:53:31] Watching paths for changes: 'docs', 'mkdocs.yml'
    INFO    -  [21:53:31] Serving on http://127.0.0.1:8000/
    ```

Content can be added now to `docs/index.md` and any new files. Changes are picked up
as soon as the files are saved to disk and the site will refresh automatically.

## Creating a Blog

[Set up a blog](https://squidfunk.github.io/mkdocs-material/setup/setting-up-a-blog/)
using the built-in `blog` plugin.

!!! info ""

    Alternatively, create a new repository with the **Use this template** button in the
    [Material for Mkdocs Blog Template](https://github.com/mkdocs-material/create-blog?tab=readme-ov-file#a-material-for-mkdocs-blog-template).

To create a blog with the built-in `blog` plugin, add it to the
`mkdocs.yml`:

``` yaml title="mkdocs.yml" hl_lines="2 3"
site_name: Inadvisably Applied Magic
plugins:
  - blog
theme:
  name: material
```

Now, as soon as the `blog` plugin is loaded, the server crashes and refuses
to start again, because the  `paginate` module cannot be found:

!!! failure "`$ mkdocs serve`"

    ``` console
    ...
      File "/usr/lib/python3/dist-packages/material/plugins/blog/plugin.py", line 38, in <module>
        from paginate import Page as Pagination
    ModuleNotFoundError: No module named 'paginate'
    ```

There is no system package in Ubuntu 24.04 that provides this module,
so it must be installed via `pip` (forced with `--break-system-packages`):

??? terminal "`$ sudo pip3 install --break-system-packages paginate`"

    ``` console
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
e.g. `docs/blog/posts/2024-11-01-this-is-only-a-test.md`, but the `date` in the
metadata will take precedence over that in the file name.

``` markdown title="2024-11-01-this-is-only-a-test.md"
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

``` markdown title="2024-11-01-this-is-only-a-test.md"
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
[Continuous Monitoring](../../projects/conmon.md).

It can also be good to have a clear separation between the blog,
as an otherwise unorganized timeline of events, and the more
(if not better) organized content in static pages.

### Site-agnostic URLs

To make links to other pages in the same site, and image URLs,
agnostic of where the site is hostead, and whether the blog is
using the `same-dir` plugin or not, `extra` data can be added to
the `mkdocs.yml` configuration and used in content generation via
the `macros` plugin:

``` yaml title="mkdocs.yml" hl_lines="1-3 6-7"
extra:
  site:
    baseurl: http://127.0.0.1:8000/
plugins:
  - blog
  - macros:
      on_undefined: strict
site_name: Inadvisably Applied Magic
```

This may require installing the `macros` pluging in the system;
in Ubuntu this requires installing `mkdocs-macros-plugin`:

??? terminal "`# sudo apt install mkdocs-macros-plugin`"

    ``` console
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

### GLightbox

[MkDocs GLightbox](https://github.com/blueswen/mkdocs-glightbox?tab=readme-ov-file#mkdocs-glightbox)
is a nice plugin that supports image lightbox with GLightbox,
a pure javascript lightbox library with mobile support.

Again, there is no system package in Ubuntu 24.04 so it must be installed via `pip`.

??? terminal "`$ sudo pip3 install --break-system-packages mkdocs-glightbox`"

    ``` console
    $ sudo pip install --break-system-packages mkdocs-glightbox
    Collecting mkdocs-glightbox
      Downloading mkdocs_glightbox-0.4.0-py3-none-any.whl.metadata (6.1 kB)
    Downloading mkdocs_glightbox-0.4.0-py3-none-any.whl (31 kB)
    Installing collected packages: mkdocs-glightbox
    Successfully installed mkdocs-glightbox-0.4.0
    WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv
    ``` 

## Publish to GitHub Pages

To publish the new site to GitHub Pages, replacing the previous
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

!!! tip
    It would have been enough to simply create the `.github/workflows/ci.yml`
    file and the "Build and deployment" settings under "Source" as they were,
    so long as the selected branch is **gh-pages** (and the forder is **/ (root)**).

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

``` console
ERROR   -  Config value 'plugins': The "macros" plugin is not installed
``` 

To fix this, add a step to install the plugin in the
`.github/workflows/ci.yml` just before deploying:

``` yaml title=".github/workflows/ci.yml"
      - run: pip install mkdocs-material 
      - run: pip install mkdocs-macros-plugin
      - run: pip install mkdocs-glightbox
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

``` console
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

``` console
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

!!! tip
    Instead of the password, use a
    [fine-grain token for personal access](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token).

## Final Tweaks

Many features were added to `mkdocs.yml` during the above process, such as adding
[Google Analytics](https://squidfunk.github.io/mkdocs-material/setup/setting-up-site-analytics/#google-analytics),
adding
[a Dark / Light themes switch](https://yodamad.hashnode.dev/beautiful-pages-on-gitlab-with-mkdocs#heading-dark-light-themes-switch),
changing the colors and logo, etc. By the times this post was finished,
this is what `mkdocs.yml` looked like:

``` yaml linenums="1" title="mkdocs.yml"
copyright: Copyright &copy; 2019-2025 Ponder Stibbons
extra:
  analytics:
    provider: google
    property: G-Y167L3ZVGY
extra_css:
  - stylesheets/extra.css
markdown_extensions:
  - admonition
  - attr_list
  - md_in_html
  - pymdownx.details
  - pymdownx.emoji:
      emoji_index: !!python/name:material.extensions.emoji.twemoji
      emoji_generator: !!python/name:material.extensions.emoji.to_svg
  - pymdownx.highlight:
      anchor_linenums: true
      line_spans: __span
      pygments_lang_class: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences
  - pymdownx.tabbed:
      alternate_style: true
nav:
  - About This Place: 'index.md'
  - Continuous Monitoring: 'conmon.md'
  - Unexpected Linux Adventures: 'blog/index.md'
plugins:
  - blog:
      draft: false
      draft_on_serve: false
      pagination: true
      pagination_per_page: 10
  - macros:
      on_undefined: strict
site_name: Inadvisably Applied Magic
site_url: https://stibbons1990.github.io/hex/
theme:
  name: material
  features:
    - content.code.copy
  icon:
    logo: fontawesome/solid/hat-wizard
  palette:
    # Dark mode
    - scheme: slate
      primary: deep purple
      accent: green
      toggle:
        icon: material/weather-sunny
        name: Dark mode
    # Light mode
    - scheme: default
      primary: deep purple
      accent: green
      toggle:
        icon: material/weather-night
        name: Light mode
```

### Custom Admonitions

This blog contains many long sections capturing the output from running shell
commands in a terminal emulator. These are often long, even *very long*, and
most of the times need not be read, or even visible by default.

There are also long samples of source code and configuration files, such as
Xorg, Systemd, Kubernetes and [many more languages](https://pygments.org/languages/)
supported by [Pygments](https://pygments.org/).

To show these blocks differently, a custom `terminal` admonitions is used,
based on the `pied-piper` example in
[Custom admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/#custom-admonitions),
using the SVG from the :fontawesome-solid-terminal: emoji for the icon:

``` css linenums="1" title="stylesheets/extra.css"
.md-typeset .admonition,
.md-typeset details {
  font-size: 16px
}
:root {
    --md-admonition-icon--terminal: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 576 512"><path d="M9.4 86.6c-12.5-12.5-12.5-32.7 0-45.2s32.8-12.5 45.3 0l192 192c12.5 12.5 12.5 32.8 0 45.3l-192 192c-12.5 12.5-32.8 12.5-45.3 0s-12.5-32.8 0-45.3L178.7 256 9.4 86.6zM256 416h288c17.7 0 32 14.3 32 32s-14.3 32-32 32H256c-17.7 0-32-14.3-32-32s14.3-32 32-32z"/></svg>')
  }
  .md-typeset .admonition.terminal,
  .md-typeset details.terminal {
    border-color: rgb(43, 155, 70);
  }
  .md-typeset .terminal > .admonition-title,
  .md-typeset .terminal > summary {
    background-color: rgba(43, 155, 70, 0.1);
  }
  .md-typeset .terminal > .admonition-title::before,
  .md-typeset .terminal > summary::before {
    background-color: rgb(43, 155, 70);
    -webkit-mask-image: var(--md-admonition-icon--terminal);
            mask-image: var(--md-admonition-icon--terminal);
  }
```

The admonition can be collapsed by default when the output is long.
One down-side is that it's hard to copy the title of the admonition
(the command to run), it is much easier to copy it from either the
content (inside the admonition) and/or a separate code block
*outside* the admonition.

??? terminal "`$ date -R`"

    ``` console
    $ date -R
    Fri, 13 Dec 2024 23:05:53 +0100
    ```

The first 4 lines in the above CSS
[change the font-size of the admonition block](https://github.com/squidfunk/mkdocs-material/discussions/2260)
so that the text, output or code is more easily readable.

!!! news

    More custom admonitions are added as the need turms up,
    what follows is the full `extra.css` file with all of these.

??? code "Here goes the entire [`extra.css`](../../stylesheets/extra.css) file:"

    ``` css linenums="1" title="stylesheets/extra.css"
    .md-typeset .admonition,
    .md-typeset details {
      font-size: 16px
    }

    :root {
        --md-admonition-icon--terminal: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 576 512"><path d="M9.4 86.6c-12.5-12.5-12.5-32.7 0-45.2s32.8-12.5 45.3 0l192 192c12.5 12.5 12.5 32.8 0 45.3l-192 192c-12.5 12.5-32.8 12.5-45.3 0s-12.5-32.8 0-45.3L178.7 256 9.4 86.6zM256 416h288c17.7 0 32 14.3 32 32s-14.3 32-32 32H256c-17.7 0-32-14.3-32-32s14.3-32 32-32z"/></svg>')
      }
      .md-typeset .admonition.terminal,
      .md-typeset details.terminal {
        border-color: rgb(43, 155, 70);
      }
      .md-typeset .terminal > .admonition-title,
      .md-typeset .terminal > summary {
        background-color: rgba(43, 155, 70, 0.1);
      }
      .md-typeset .terminal > .admonition-title::before,
      .md-typeset .terminal > summary::before {
        background-color: rgb(43, 155, 70);
        -webkit-mask-image: var(--md-admonition-icon--terminal);
                mask-image: var(--md-admonition-icon--terminal);
      }

    :root {
        --md-admonition-icon--todo: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M11 9h2V7h-2m1 13c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8m0-18A10 10 0 0 0 2 12a10 10 0 0 0 10 10 10 10 0 0 0 10-10A10 10 0 0 0 12 2m-1 15h2v-6h-2v6Z"/></svg>')
      }
      .md-typeset .admonition.todo,
      .md-typeset details.todo {
        border-color: rgb(188, 128, 128);
      }
      .md-typeset .todo > .admonition-title,
      .md-typeset .todo > summary {
        background-color: rgba(188, 128, 128, 0.1);
      }
      .md-typeset .todo > .admonition-title::before,
      .md-typeset .todo > summary::before {
        background-color: rgb(188, 128, 128);
        -webkit-mask-image: var(--md-admonition-icon--todo);
                mask-image: var(--md-admonition-icon--todo);
      }

    :root {
        --md-admonition-icon--news: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M20 11H4V8h16m0 7h-7v-2h7m0 6h-7v-2h7m-9 2H4v-6h7m9.33-8.33L18.67 3 17 4.67 15.33 3l-1.66 1.67L12 3l-1.67 1.67L8.67 3 7 4.67 5.33 3 3.67 4.67 2 3v16a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V3l-1.67 1.67Z"/></svg>')
      }
      .md-typeset .admonition.news,
      .md-typeset details.news {
        border-color: rgb(128, 128, 188);
      }
      .md-typeset .news > .admonition-title,
      .md-typeset .news > summary {
        background-color: rgba(128, 128, 188, 0.1);
      }
      .md-typeset .news > .admonition-title::before,
      .md-typeset .news > summary::before {
        background-color: rgb(128, 128, 188);
        -webkit-mask-image: var(--md-admonition-icon--news);
                mask-image: var(--md-admonition-icon--news);
      }

    :root {
        --md-admonition-icon--k8s: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="m10.204 14.35.007.01-.999 2.413a5.171 5.171 0 0 1-2.075-2.597l2.578-.437.004.005a.44.44 0 0 1 .484.606zm-.833-2.129a.44.44 0 0 0 .173-.756l.002-.011L7.585 9.7a5.143 5.143 0 0 0-.73 3.255l2.514-.725.002-.009zm1.145-1.98a.44.44 0 0 0 .699-.337l.01-.005.15-2.62a5.144 5.144 0 0 0-3.01 1.442l2.147 1.523.004-.002zm.76 2.75.723.349.722-.347.18-.78-.5-.623h-.804l-.5.623.179.779zm1.5-3.095a.44.44 0 0 0 .7.336l.008.003 2.134-1.513a5.188 5.188 0 0 0-2.992-1.442l.148 2.615.002.001zm10.876 5.97-5.773 7.181a1.6 1.6 0 0 1-1.248.594l-9.261.003a1.6 1.6 0 0 1-1.247-.596l-5.776-7.18a1.583 1.583 0 0 1-.307-1.34L2.1 5.573c.108-.47.425-.864.863-1.073L11.305.513a1.606 1.606 0 0 1 1.385 0l8.345 3.985c.438.209.755.604.863 1.073l2.062 8.955c.108.47-.005.963-.308 1.34zm-3.289-2.057c-.042-.01-.103-.026-.145-.034-.174-.033-.315-.025-.479-.038-.35-.037-.638-.067-.895-.148-.105-.04-.18-.165-.216-.216l-.201-.059a6.45 6.45 0 0 0-.105-2.332 6.465 6.465 0 0 0-.936-2.163c.052-.047.15-.133.177-.159.008-.09.001-.183.094-.282.197-.185.444-.338.743-.522.142-.084.273-.137.415-.242.032-.024.076-.062.11-.089.24-.191.295-.52.123-.736-.172-.216-.506-.236-.745-.045-.034.027-.08.062-.111.088-.134.116-.217.23-.33.35-.246.25-.45.458-.673.609-.097.056-.239.037-.303.033l-.19.135a6.545 6.545 0 0 0-4.146-2.003l-.012-.223c-.065-.062-.143-.115-.163-.25-.022-.268.015-.557.057-.905.023-.163.061-.298.068-.475.001-.04-.001-.099-.001-.142 0-.306-.224-.555-.5-.555-.275 0-.499.249-.499.555l.001.014c0 .041-.002.092 0 .128.006.177.044.312.067.475.042.348.078.637.056.906a.545.545 0 0 1-.162.258l-.012.211a6.424 6.424 0 0 0-4.166 2.003 8.373 8.373 0 0 1-.18-.128c-.09.012-.18.04-.297-.029-.223-.15-.427-.358-.673-.608-.113-.12-.195-.234-.329-.349a2.691 2.691 0 0 0-.111-.088.594.594 0 0 0-.348-.132.481.481 0 0 0-.398.176c-.172.216-.117.546.123.737l.007.005.104.083c.142.105.272.159.414.242.299.185.546.338.743.522.076.082.09.226.1.288l.16.143a6.462 6.462 0 0 0-1.02 4.506l-.208.06c-.055.072-.133.184-.215.217-.257.081-.546.11-.895.147-.164.014-.305.006-.48.039-.037.007-.09.02-.133.03l-.004.002-.007.002c-.295.071-.484.342-.423.608.061.267.349.429.645.365l.007-.001.01-.003.129-.029c.17-.046.294-.113.448-.172.33-.118.604-.217.87-.256.112-.009.23.069.288.101l.217-.037a6.5 6.5 0 0 0 2.88 3.596l-.09.218c.033.084.069.199.044.282-.097.252-.263.517-.452.813-.091.136-.185.242-.268.399-.02.037-.045.095-.064.134-.128.275-.034.591.213.71.248.12.556-.007.69-.282v-.002c.02-.039.046-.09.062-.127.07-.162.094-.301.144-.458.132-.332.205-.68.387-.897.05-.06.13-.082.215-.105l.113-.205a6.453 6.453 0 0 0 4.609.012l.106.192c.086.028.18.042.256.155.136.232.229.507.342.84.05.156.074.295.145.457.016.037.043.09.062.129.133.276.442.402.69.282.247-.118.341-.435.213-.71-.02-.039-.045-.096-.065-.134-.083-.156-.177-.261-.268-.398-.19-.296-.346-.541-.443-.793-.04-.13.007-.21.038-.294-.018-.022-.059-.144-.083-.202a6.499 6.499 0 0 0 2.88-3.622c.064.01.176.03.213.038.075-.05.144-.114.28-.104.266.039.54.138.87.256.154.06.277.128.448.173.036.01.088.019.13.028l.009.003.007.001c.297.064.584-.098.645-.365.06-.266-.128-.537-.423-.608zM16.4 9.701l-1.95 1.746v.005a.44.44 0 0 0 .173.757l.003.01 2.526.728a5.199 5.199 0 0 0-.108-1.674A5.208 5.208 0 0 0 16.4 9.7zm-4.013 5.325a.437.437 0 0 0-.404-.232.44.44 0 0 0-.372.233h-.002l-1.268 2.292a5.164 5.164 0 0 0 3.326.003l-1.27-2.296h-.01zm1.888-1.293a.44.44 0 0 0-.27.036.44.44 0 0 0-.214.572l-.003.004 1.01 2.438a5.15 5.15 0 0 0 2.081-2.615l-2.6-.44-.004.005z"/></svg>')
      }
      .md-typeset .admonition.k8s,
      .md-typeset details.k8s {
        border-color: rgb(61, 109, 223);
      }
      .md-typeset .k8s > .admonition-title,
      .md-typeset .k8s > summary {
        background-color: rgba(61, 109, 223, 0.1);
      }
      .md-typeset .k8s > .admonition-title::before,
      .md-typeset .k8s > summary::before {
        background-color: rgb(61, 109, 223);
        -webkit-mask-image: var(--md-admonition-icon--k8s);
                mask-image: var(--md-admonition-icon--k8s);
      }

    :root {
        --md-admonition-icon--code: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M5.59 3.41L7 4.82L3.82 8L7 11.18L5.59 12.6L1 8L5.59 3.41M11.41 3.41L16 8L11.41 12.6L10 11.18L13.18 8L10 4.82L11.41 3.41M22 6V18C22 19.11 21.11 20 20 20H4C2.9 20 2 19.11 2 18V14H4V18H20V6H17.03V4H20C21.11 4 22 4.89 22 6Z"/></svg>')
      }
      .md-typeset .admonition.code,
      .md-typeset details.code {
        border-color: rgb(43, 155, 70);
      }
      .md-typeset .code > .admonition-title,
      .md-typeset .code > summary {
        background-color: rgba(43, 155, 70, 0.1);
      }
      .md-typeset .code > .admonition-title::before,
      .md-typeset .code > summary::before {
        background-color: rgb(43, 155, 70);
        -webkit-mask-image: var(--md-admonition-icon--code);
                mask-image: var(--md-admonition-icon--code);
      }

    :root {
        --md-admonition-icon--json: url('data:image/svg+xml;charset=utf-8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M5 3C3.9 3 3 3.9 3 5S2.1 7 1 7V9C2.1 9 3 9.9 3 11S3.9 13 5 13H7V11H5V10C5 8.9 4.1 8 3 8C4.1 8 5 7.1 5 6V5H7V3M11 3C12.1 3 13 3.9 13 5S13.9 7 15 7V9C13.9 9 13 9.9 13 11S12.1 13 11 13H9V11H11V10C11 8.9 11.9 8 13 8C11.9 8 11 7.1 11 6V5H9V3H11M22 6V18C22 19.11 21.11 20 20 20H4C2.9 20 2 19.11 2 18V15H4V18H20V6H17.03V4H20C21.11 4 22 4.89 22 6Z"/></svg>')
      }
      .md-typeset .admonition.json,
      .md-typeset details.json {
        border-color: rgb(199, 146, 234);
      }
      .md-typeset .json > .admonition-title,
      .md-typeset .json > summary {
        background-color: rgba(199, 146, 234, 0.1);
      }
      .md-typeset .json > .admonition-title::before,
      .md-typeset .json > summary::before {
        background-color: rgb(199, 146, 234);
        -webkit-mask-image: var(--md-admonition-icon--json);
                mask-image: var(--md-admonition-icon--json);
      }
    ```

!!! note

    The `<svg>` tags in each `--md-admonition-icon` URL are the result of adding
    [Icons, Emojis](https://squidfunk.github.io/mkdocs-material/reference/icons-emojis/)
    to a page, then grabbing the `<svg>` HTML element using browser inspection tools.

!!! todo "Find a way to embed the `extra.css` file directly."
