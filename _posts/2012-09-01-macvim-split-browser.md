---
layout:    post
title:    "Installing MacVIM with split browser support using homebrew"
date:     2012-09-01
comments: true
categories: tools howto
---

I'm a old-school coder and my first editor was *vi* on a old terminal
hooked to some big-iron unix box.  I never recovered -- thus I'm still
using a modal editor -- [MacVIM] [1].

MacVIM is a [Vim] [2] port for OSX with quite some extensions like multiple
windows, editor tabs, nice font rendering etc.

There's also a experimental branch which adds a file-browser side pane, which is
implemented in Objective-C and thus very fast.

![MacVIM split browser](/assets/images/macvim-split-browser.jpg)

Keeping up-to-date with this branch is tedious -- cloning, pulling in new sources,
compiling ...

Today I accidentally found the [homebrew recipe] [3] from Joel Cogen -- and in his
read-me a feature of homebrew which was new to me -- `brew tap <<github repo name>>`.

<!-- more -->

So you do a:

```bash

$ brew tap joelcogen/macvimsplitbrowser
Cloning into '/usr/local/Library/Taps/joelcogen-macvimsplitbrowser'...
remote: Counting objects: 7, done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 7 (delta 1), reused 7 (delta 1)
Unpacking objects: 100% (7/7), done.
Tapped 1 formula
```

And [homebrew] [4] will happily clone the repository specified to `/usr/local/Library/Taps`.  This
means that I'm now able to install MacVIM with split browser support using homebrew:

```bash
$ brew install macvim-split-browser
==> Downloading https://github.com/alloy/macvim/tarball/1167a32988202c3c81842371ecf9f80fe4854172
######################################################################## 100,0%
==> ./configure --with-features=huge --with-tlib=ncurses --enable-multibyte --with-macarchs=x86_64 --e
==> cd src/MacVim/icons && make getenvy
==> make
==> Caveats
This formula will most probably conflict with the macvim formula.
If you already installed MacVim from Homebrew, please remove it using:
    brew uninstall macvim

MacVim.app installed to:
  /usr/local/Cellar/macvim-split-browser/20120109

To link the application to a normal Mac OS X location:
    brew linkapps
or:
    ln -s /usr/local/Cellar/macvim-split-browser/20120109/MacVim.app /Applications
==> Summary
/usr/local/Cellar/macvim-split-browser/20120109: 1730 files, 26M, built in 48 seconds
```

Installing links in `~/Applications` is done as usual:

```bash
osx/macvim [ brew linkapps                                                   split-browser ] 12:25 pm
Linking /usr/local/Cellar/macvim-split-browser/20120109/MacVim.app
Finished linking. Find the links under ~/Applications.
```

[1]:  http://macvim.org
[2]:  http://vim.org
[3]:  https://github.com/joelcogen/homebrew-macvimsplitbrowser
[4]:  https://github.com/homebrew
