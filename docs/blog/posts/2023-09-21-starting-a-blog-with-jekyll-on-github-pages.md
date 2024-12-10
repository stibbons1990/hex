---
date: 2023-09-21
categories:
 - github
 - jekyll
 - markdown
 - blog
 - migration
title: Starting a blog with Jekyll on GitHub pages
---

Over the last couple of weekends I've been trying a couple of blogging platforms,
namely [WordPress.com](https://wordpress.com) and [Blogger](https://blogger.com).

<!-- more --> 

Each have their pros and cons, but to cut a story short:

*  [WordPress.com](https://wordpress.com) is quick’n’easy to setup, 
   but the editor gets painfully slow with long (and not really all that long) articles, 
   themes are very limited and can’t be customized.
   *  On the plus side, its *code highlight* block is quite neat.
*  [Blogger](https://blogger.com) is also quick’n’easy to setup, the editor works well enough,
   allows editing most of the content as HTML and then uploading images *and videos*,
   and themes are fully customizable (can be edited raw).
   *  On the **huge** downside, it will unpublish, block or remove posts *even entire blogs*,
      and there seems to be no way to appeal.

## Setup

Jekyll on GitHub pages seems like a much better option, specially since Markdown offers a
good balance between easy-to-write and *close enough* to HTML.

Dang, I can even see *italic* and **bold** text in VIM!

Anyway, setup was also *relatively* quick'n'easy:

1. Create public repository
   [stibbons1990/hex][stibbons1990/hex].
2. Follow [docs.github.com/en/pages/quickstart](https://docs.github.com/en/pages/quickstart)
   to publish the main branch to 
   [stibbons1990.github.io/hex][stibbons1990.io/hex].
3. [Install Jekyll on Ubuntu][jekyllrb-installation-ubuntu] (including Ruby):
   ```
   $ sudo apt-get install ruby-full build-essential zlib1g-dev
   $ echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
   $ echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
   $ echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
   $ source ~/.bashrc
   $ gem install jekyll bundler
   ...
   Done installing documentation for bundler after 0 seconds
   28 gems installed
   ```
4. Follow up to steps **7** trough 13 (1-6 involve the creation a repo, already done) of [Creating your site][creating-your-site]:
   ```
   $ jekyll new --skip-bundle . --force
   New jekyll site installed in /home/k8s/code-server/hex. 
   Bundle install skipped. 
   
   $ bundle install
   Fetching gem metadata from https://rubygems.org/............
   Resolving dependencies...
   Fetching webrick 1.8.1
   Fetching sass-embedded 1.68.0
   Installing webrick 1.8.1
   Installing sass-embedded 1.68.0 with native extensions
   Fetching jekyll-feed 0.17.0
   Installing jekyll-feed 0.17.0
   Bundle complete! 7 Gemfile dependencies, 34 gems now installed.
   Use `bundle info [gemname]` to see where a bundled gem is installed.
   ```

5. Commit the changes and uush the files to GitHub
   ```
   $ git add .
   $ git commit -m "bundle install" -a
   $ git push
   ```

6. After a few minutes, the new blog is live at
   [stibbons1990.github.io/hex][stibbons1990.io/hex].

**Notes**:
   *  After `git push` it takes a few minutes for GitHub to generate the
      content. Check out the latest [actions](https://github.com/stibbons1990/hex/actions/) to see its progress.
   *  If the new repository has *any* files in it (e.g `LICENSE`),
     `jekyll` will report a conflict and ask that the directory be empty
     *or else try again with `--force` to proceed and overwrite any files.*

## Local Testing on PC

Testing locally on the PC was a little rockier, the first time failed with:

```
$ bundle exec jekyll serve
Configuration file: /home/k8s/code-server/hex/_config.yml
To use retry middleware with Faraday v2.0+, install `faraday-retry` gem
            Source: /home/k8s/code-server/hex
       Destination: /home/k8s/code-server/hex/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
                    done in 0.172 seconds.
 Auto-regeneration: enabled for '/home/k8s/code-server/hex'
bundler: failed to load command: jekyll (/home/k8s/code-server/gems/bin/jekyll)
/home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve/servlet.rb:3:in `require': cannot load such file -- webrick (LoadError)
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve/servlet.rb:3:in `<top (required)>'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:184:in `require_relative'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:184:in `setup'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:102:in `process'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:93:in `block in start'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:93:in `each'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:93:in `start'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/lib/jekyll/commands/serve.rb:75:in `block (2 levels) in init_with_program'
        from /home/k8s/code-server/gems/gems/mercenary-0.3.6/lib/mercenary/command.rb:220:in `block in execute'
        from /home/k8s/code-server/gems/gems/mercenary-0.3.6/lib/mercenary/command.rb:220:in `each'
        from /home/k8s/code-server/gems/gems/mercenary-0.3.6/lib/mercenary/command.rb:220:in `execute'
        from /home/k8s/code-server/gems/gems/mercenary-0.3.6/lib/mercenary/program.rb:42:in `go'
        from /home/k8s/code-server/gems/gems/mercenary-0.3.6/lib/mercenary.rb:19:in `program'
        from /home/k8s/code-server/gems/gems/jekyll-3.9.3/exe/jekyll:15:in `<top (required)>'
        from /home/k8s/code-server/gems/bin/jekyll:25:in `load'
        from /home/k8s/code-server/gems/bin/jekyll:25:in `<top (required)>'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/cli/exec.rb:58:in `load'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/cli/exec.rb:58:in `kernel_load'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/cli/exec.rb:23:in `run'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/cli.rb:492:in `exec'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/vendor/thor/lib/thor/command.rb:27:in `run'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/vendor/thor/lib/thor/invocation.rb:127:in `invoke_command'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/vendor/thor/lib/thor.rb:392:in `dispatch'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/cli.rb:34:in `dispatch'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/vendor/thor/lib/thor/base.rb:485:in `start'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/cli.rb:28:in `start'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/exe/bundle:37:in `block in <top (required)>'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/lib/bundler/friendly_errors.rb:117:in `with_friendly_errors'
        from /home/k8s/code-server/gems/gems/bundler-2.4.19/exe/bundle:29:in `<top (required)>'
        from /home/k8s/code-server/gems/bin/bundle:25:in `load'
        from /home/k8s/code-server/gems/bin/bundle:25:in `<main>'
```

Workaround from [github.com/jekyll/jekyll/issues/8523][jekyll-issues-8523] helped:

```
$ echo 'gem "webrick"' >> Gemfile
$ bundle exec jekyll serve
Configuration file: /home/k8s/code-server/hex/_config.yml
To use retry middleware with Faraday v2.0+, install `faraday-retry` gem
            Source: /home/k8s/code-server/hex
       Destination: /home/k8s/code-server/hex/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
                    done in 0.164 seconds.
 Auto-regeneration: enabled for '/home/k8s/code-server/hex'
    Server address: http://127.0.0.1:4000/hex/
  Server running... press ctrl-c to stop.
```

And this time it worked wonderfully, I couldn't tell the difference between the local site
and the live site once deployed.

## Local Testing on Server

Testing on the server (Lexicon) as a little bumpty to;
the first time running the server didn't fail,
but the server wouldn't respond to external hosts.
The trick to make this work is adding `--host 0.0.0.0`
so that the web server listens on all network interfaces:

```
$ bundle exec jekyll serve --host 0.0.0.0
Configuration file: /home/k8s/code-server/hex/_config.yml
To use retry middleware with Faraday v2.0+, install `faraday-retry` gem
            Source: /home/k8s/code-server/hex
       Destination: /home/k8s/code-server/hex/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
                    done in 0.16 seconds.
 Auto-regeneration: enabled for '/home/k8s/code-server/hex'
    Server address: http://0.0.0.0:4000/hex/
  Server running... press ctrl-c to stop.
```

From there it's up to the local network's DNS (or your
local `/etc/hosts` files) to resolve a handy hostname,
e.g. http://lexicon:4000/hex/

## Adding Themes

[Adding a theme][adding-a-theme] turned out even trickier, in
fact did not figure out how to add the first theme I tried.

First I tried adding the [Midnight theme][midnight] and,
following its instructions, got everything  messed up; the
home page was empty and the one blog post had *no style*.

Testing the site locally shows the required layouts (`home` and `post`) *don't exist*:

```
     Build Warning: Layout 'post' requested in _posts/2023-09-21-migrated-to-jekyll-on-github-pages.markdown does not exist.
   GitHub Metadata: No GitHub API authentication could be found. Some fields may be missing or have incorrect data.
     Build Warning: Layout 'page' requested in about.markdown does not exist.
     Build Warning: Layout 'home' requested in index.markdown does not exist.
```

And indeed these layouts do not exist under the
[pages-themes/midnight/_layouts](https://github.com/pages-themes/midnight/tree/master/_layouts)
directory.

Second, tried another theme found while searching fo
solutions for the problem above:
[minimal-mistakes](https://github.com/mmistakes/minimal-mistakes)

It requires [jekyll-include-cache](https://github.com/benbalter/jekyll-include-cache)
so first installed that, by adding the required lines to
[`Gemfile`](https://github.com/stibbons1990/hex/blob/main/Gemfile) and
[`_config.yml`](https://github.com/stibbons1990/hex/blob/main/_config.yml) and then running `bundle` to install it:

```
$ bundle
Fetching gem metadata from https://rubygems.org/.........
Resolving dependencies...
Fetching minimal-mistakes-jekyll 4.24.0
Installing minimal-mistakes-jekyll 4.24.0
Bundle complete! 12 Gemfile dependencies, 91 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
```

At this point following the instructions in the
[remote-theme-method](https://github.com/mmistakes/minimal-mistakes#remote-theme-method) the following should do:

In `_config.yml`

```yml
remote_theme: "mmistakes/minimal-mistakes@4.24.0"
plugins:
  - jekyll-feed
  - jekyll-remote-theme
  - jekyll-include-cache
```

In `Gemfile`
```ruby
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.12"
  gem "jekyll-remote-theme"
  gem "minimal-mistakes-jekyll"
end
```

Based on the layouts available under the
[minimal-mistakes/_layouts](https://github.com/mmistakes/minimal-mistakes/tree/master/_layouts)
directory, it seems blog posts should use `layout: single`
while other pages should use `layout: default`.

After updating those `layout` values, local testing works:

```
$ bundle exec jekyll serve --host 0.0.0.0
Configuration file: /home/k8s/code-server/hex/_config.yml
      Remote Theme: Using theme mmistakes/minimal-mistakes
To use retry middleware with Faraday v2.0+, install `faraday-retry` gem
            Source: /home/k8s/code-server/hex
       Destination: /home/k8s/code-server/hex/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
      Remote Theme: Using theme mmistakes/minimal-mistakes
       Jekyll Feed: Generating feed for posts
   GitHub Metadata: No GitHub API authentication could be found. Some fields may be missing or have incorrect data.
                    done in 1.437 seconds.
 Auto-regeneration: enabled for '/home/k8s/code-server/hex'
    Server address: http://0.0.0.0:4000/hex/
  Server running... press ctrl-c to stop.
```

Finally, we can tweak the theme's [configuration](https://mmistakes.github.io/minimal-mistakes/docs/configuration/), e.g. adding to `_config.yml` the following line to set a
different *skin*:

```yml
minimal_mistakes_skin: "neon"
```


[stibbons1990/hex]: https://github.com/stibbons1990/hex
[stibbons1990.io/hex]: https://stibbons1990.github.io/hex
[creating-a-github-pages-site-with-jekyll]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll
[jekyllrb-installation-ubuntu]: https://jekyllrb.com/docs/installation/ubuntu/
[creating-your-site]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site
[jekyll-issues-8523]: https://github.com/jekyll/jekyll/issues/8523
[adding-a-theme]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/adding-a-theme-to-your-github-pages-site-using-jekyll#adding-a-theme
[midnight]: https://github.com/pages-themes/midnight

### Customizing Minimal Mistakes

I quite like this theme as it is, but I wish it used more space
horizontally. Oddly enough, when the browser window is wider than
1280px the font size increases but the content width does not,
resulting in narrower columns and, more annoyingly, more of the
content in code blocks is hidden under the right-hand side so you
have to scroll horizontally to see it. Not very convenient.

One would expect it to be *easy* to change such a simple attribute.
And it is... *if* you read the 
[Overriding Theme Defaults](https://mmistakes.github.io/minimal-mistakes/docs/overriding-theme-defaults/)
and
[Stylesheets](https://mmistakes.github.io/minimal-mistakes/docs/stylesheets/)
pages *carefully*. If you don't, you might easily waste a few hours
going through a number of issue reports and workarounds, most of which are at least a few years old and none of which worked for me.

After reading those pages carefully, it seems the trick was to copy
the *entire*
[mmistakes/minimal-mistakes/assets/css/main.scss](https://github.com/mmistakes/minimal-mistakes/blob/master/assets/css/main.scss)
file under the project folder, as
`assets/css/main.scss`, then add the `$max-width` variable override:

```scss
---
# Only the main Sass file needs front matter (the dashes are enough)
search: false
---

@charset "utf-8";

$max-width: 1900px;

@import "minimal-mistakes/skins/{{% site.minimal_mistakes_skin | default: 'default' %}}"; // skin
@import "minimal-mistakes"; // main partials
```

#### Install Google Analytics

It *is* as easy as following Jekyll's
[Analytics](https://mmistakes.github.io/minimal-mistakes/docs/configuration/#analytics)
instructions to add the G-tag to `_config.yml`
and, assuming Google Analytics itself has been
already set up, it should *just work*.
