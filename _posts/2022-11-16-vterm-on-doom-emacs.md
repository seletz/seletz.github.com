---
layout:    post
title:    "Getting the EMACS VTerm module to work in DOOM Emacs"
date:     2022-11-16
comments: true
tags: emacs mac tools
---

[VTerm](https://duckduckgo.com/?q=emacs-vterm&t=osx) won't work OOTB on my emacs 28 with byte compilation, presumably because compilation of the `emacs-vterm` module is hacked around in the doom vterm module.

Emacs complains that It can't find the vterm module.

So we need to build that module manually:

```bash
$ brew install libvterm
$ cd develip/emacs
$ git clone https://github.com/akermu/emacs-libvterm.git
$ cd emacs-libvterm
$ mkdir -p build
$ cd build
$ cmake ..
$ make
```

Now we need to tell emacs how to find the vterm module.  I added this on the top of my `~/.doom.d/config.el`:

```elisp
(add-to-list 'load-path "~/develop/emacs/emacs-libvterm")
```

After this, `vterm` works.  Toggling the terminal is `SPC o t`.
