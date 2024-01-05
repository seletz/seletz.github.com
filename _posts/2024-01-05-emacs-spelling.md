---
layout:     post
title:      "German Spellcheck in EMACS"
date:       2024-01-05
comments:   true
tags:       emacs
---

## The problem

When entering `flyspell-mode` my EMACS complains that it can't find any
dictionaries.

Emacs uses `hunspell` for spelling checks, and it appears that I have `hunspell`
installed, but I have no dictionaries:

```shell
$ hunspell -D
SEARCH PATH:
.::/usr/share/hunspell:/usr/share/myspell:/usr/share/myspell/dicts:/Library/Spelling:/Users/seletz/.openoffice.org/3/user/wordbook:/Users/seletz/.openoffice.org2/user/wordbook:/Users/seletz/.openoffice.org2.0/user/wordbook:/Users/seletz/Library/Spelling:/opt/openoffice.org/basis3.0/share/dict/ooo:/usr/lib/openoffice.org/basis3.0/share/dict/ooo:/opt/openoffice.org2.4/share/dict/ooo:/usr/lib/openoffice.org2.4/share/dict/ooo:/opt/openoffice.org2.3/share/dict/ooo:/usr/lib/openoffice.org2.3/share/dict/ooo:/opt/openoffice.org2.2/share/dict/ooo:/usr/lib/openoffice.org2.2/share/dict/ooo:/opt/openoffice.org2.1/share/dict/ooo:/usr/lib/openoffice.org2.1/share/dict/ooo:/opt/openoffice.org2.0/share/dict/ooo:/usr/lib/openoffice.org2.0/share/dict/ooo
AVAILABLE DICTIONARIES (path is not mandatory for -d option):
Can't open affix or dictionary files for dictionary named "de_DE".
```

## Getting dictionaries

It's surprisingly difficult to google for where to get `hunspell` dictionaries.  There's
[this](https://www.j3e.de/ispell/igerman98/) -- it seems Björn Jacke maintains German `ispell`
dictionaries for the current German spelling rules.

The `hunnspell` package on for OSX gives a link and information on where to find
dictionaries:

```shell
$ brew info hunspell
...
==> Caveats
Dictionary files (*.aff and *.dic) should be placed in
~/Library/Spelling/ or /Library/Spelling/.  Homebrew itself
provides no dictionaries for Hunspell, but you can download
compatible dictionaries from other sources, such as
https://cgit.freedesktop.org/libreoffice/dictionaries/tree/ .
...
```

So for getting the dictionaries I finally did:

```shell
$ cd ~/Library/Spelling
$ curl --remote-name-all https://cgit.freedesktop.org/libreoffice/dictionaries/plain/en/en_\{GB,US\}.\{dic,aff\}
$ curl --remote-name-all https://cgit.freedesktop.org/libreoffice/dictionaries/plain/de/de_\{AT,DE\}_frami.\{dic,aff\}
```

Then I added some symlinks for `de_DE`:

```shell
$ ln -s de_DE_frami.aff de_DE.aff
ln -s de_AT_frami.aff de_AT.aff
$ ll
total 26192
drwx------+ 12 seletz  staff   384B  5 Jan 10:46 .
drwx------@ 97 seletz  staff   3,0K 31 Dez 09:48 ..
-rw-r--r--@  1 seletz  staff    19K  5 Jan 10:46 de_AT_frami.aff
-rw-r--r--@  1 seletz  staff   4,2M  5 Jan 10:46 de_AT_frami.dic
lrwxr-xr-x@  1 seletz  staff    15B  5 Jan 10:46 de_DE.aff -> de_DE_frami.aff
lrwxr-xr-x@  1 seletz  staff    15B  5 Jan 10:46 de_DE.dic -> de_DE_frami.dic
-rw-r--r--@  1 seletz  staff    19K  5 Jan 10:46 de_DE_frami.aff
-rw-r--r--@  1 seletz  staff   4,2M  5 Jan 10:46 de_DE_frami.dic
-rw-r--r--@  1 seletz  staff    34K  5 Jan 10:44 en_GB.aff
-rw-r--r--@  1 seletz  staff   1,2M  5 Jan 10:44 en_GB.dic
-rw-r--r--@  1 seletz  staff   3,0K  5 Jan 10:44 en_US.aff
-rw-r--r--@  1 seletz  staff   539K  5 Jan 10:44 en_US.dic
```

This silenced the `hunspell` error on the CLI:

```shell
$ hunspell -D
SEARCH PATH:
.::/usr/share/hunspell:/usr/share/myspell:/usr/share/myspell/dicts:/Library/Spelling:/Users/seletz/.openoffice.org/3/user/wordbook:/Users/seletz/.openoffice.org2/user/wordbook:/Users/seletz/.openoffice.org2.0/user/wordbook:/Users/seletz/Library/Spelling:/opt/openoffice.org/basis3.0/share/dict/ooo:/usr/lib/openoffice.org/basis3.0/share/dict/ooo:/opt/openoffice.org2.4/share/dict/ooo:/usr/lib/openoffice.org2.4/share/dict/ooo:/opt/openoffice.org2.3/share/dict/ooo:/usr/lib/openoffice.org2.3/share/dict/ooo:/opt/openoffice.org2.2/share/dict/ooo:/usr/lib/openoffice.org2.2/share/dict/ooo:/opt/openoffice.org2.1/share/dict/ooo:/usr/lib/openoffice.org2.1/share/dict/ooo:/opt/openoffice.org2.0/share/dict/ooo:/usr/lib/openoffice.org2.0/share/dict/ooo
AVAILABLE DICTIONARIES (path is not mandatory for -d option):
/Users/seletz/Library/Spelling/de_DE_frami
/Users/seletz/Library/Spelling/de_DE
/Users/seletz/Library/Spelling/en_GB
/Users/seletz/Library/Spelling/de_AT_frami
/Users/seletz/Library/Spelling/en_US
LOADED DICTIONARY:
/Users/seletz/Library/Spelling/de_DE.aff
/Users/seletz/Library/Spelling/de_DE.dic
```

## Emacs configuration

On the web I found a blog post[^blogpost], following that I added the following
to my EMACS `config.el`:

```emacs-lisp
(with-eval-after-load "ispell"
  ;; Configure `LANG`, otherwise ispell.el cannot find a 'default
  ;; dictionary' even though multiple dictionaries will be configured
  ;; in next line.
  (setenv "LANG" "en_US")
  (setq ispell-program-name "hunspell")
  ;; Configure German, Swiss German, and two variants of English.
  (setq ispell-dictionary "de_DE_frami,en_GB,en_US")
  ;; ispell-set-spellchecker-params has to be called
  ;; before ispell-hunspell-add-multi-dic will work
  (ispell-set-spellchecker-params)
  (ispell-hunspell-add-multi-dic "de_DE_frami,en_GB,en_US")
  ;; For saving words to the personal dictionary, don't infer it from
  ;; the locale, otherwise it would save to ~/.hunspell_de_DE.
  (setq ispell-personal-dictionary "~/.hunspell_personal")
  )

;; The personal dictionary file has to exist, otherwise hunspell will
;; silently not use it.
(unless (file-exists-p ispell-personal-dictionary)
  (write-region "" nil ispell-personal-dictionary nil 0))
```

## Spell checking in Emacs

To toggle `flyspell mode`, in DOOM EMACS do `SPC t s` (or `M-x flyspell-mode`).  Clicking on marked
words opens up a mini-buffer showing corrections, using `z =` does the same.

---

[^blogpost]: [Blog post](https://200ok.ch/posts/2020-08-22_setting_up_spell_checking_with_multiple_dictionaries.html) from Alain M. Lafon about setting up Fly-spell in EMACS with multiple dictionaries.
