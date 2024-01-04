---
title: German Spellcheck in EMACS
---

## The problem

When entering `flyspell-mode` my EMACS complains that it can't find any
dictionaries.

Emacs uses `hunspell` for spelling checks, and it appears that I have `hunspell`
installed, but I have no dichtionaries:

```shell
$ hunspell -D
SEARCH PATH:
.::/usr/share/hunspell:/usr/share/myspell:/usr/share/myspell/dicts:/Library/Spelling:/Users/seletz/.openoffice.org/3/user/wordbook:/Users/seletz/.openoffice.org2/user/wordbook:/Users/seletz/.openoffice.org2.0/user/wordbook:/Users/seletz/Library/Spelling:/opt/openoffice.org/basis3.0/share/dict/ooo:/usr/lib/openoffice.org/basis3.0/share/dict/ooo:/opt/openoffice.org2.4/share/dict/ooo:/usr/lib/openoffice.org2.4/share/dict/ooo:/opt/openoffice.org2.3/share/dict/ooo:/usr/lib/openoffice.org2.3/share/dict/ooo:/opt/openoffice.org2.2/share/dict/ooo:/usr/lib/openoffice.org2.2/share/dict/ooo:/opt/openoffice.org2.1/share/dict/ooo:/usr/lib/openoffice.org2.1/share/dict/ooo:/opt/openoffice.org2.0/share/dict/ooo:/usr/lib/openoffice.org2.0/share/dict/ooo
AVAILABLE DICTIONARIES (path is not mandatory for -d option):
Can't open affix or dictionary files for dictionary named "de_DE".
```

## Getting dictionaries

It's surprisingly difficult to google for where to get hunspell dictionaries.  I finally found [this](https://www.j3e.de/ispell/igerman98/) -- it seems Björn Jacke maintains german `ispell` 
dictionaries for the current german spelling rules.

> It appears, that on debian Linux this all is as simple as:
> ```shell
> $ apt install hunspell hunspell-de-de hunspell-en-us hunspell-en-gb
> ```
> Oh well.  See the links below.
{: .prompt-tip }

After downloading and unpacking, I did:

```shell
$ make ispell/de_DE.aff ispell/de_DE.hash ispell/de_AT.aff ispell/de_AT.hash ispell/de_CH.aff ispell/de_CH.hash
$ make hunspell/de_DE.aff hunspell/de_DE.dic hunspell/de_AT.aff hunspell/de_AT.dic hunspell/de_CH.aff hunspell/de_CH.dic
```

This took a while, then:

```shell
$ cp hunspell/*.dic ~/Library/Spelling
$ cp hunspell/*.aff ~/Library/Spelling
```

Now `hunspell` does no longer complain:

```shell
❯ hunspell -D
SEARCH PATH:
.::/usr/share/hunspell:/usr/share/myspell:/usr/share/myspell/dicts:/Library/Spelling:/Users/seletz/.openoffice.org/3/user/wordbook:/Users/seletz/.openoffice.org2/user/wordbook:/Users/seletz/.openoffice.org2.0/user/wordbook:/Users/seletz/Library/Spelling:/opt/openoffice.org/basis3.0/share/dict/ooo:/usr/lib/openoffice.org/basis3.0/share/dict/ooo:/opt/openoffice.org2.4/share/dict/ooo:/usr/lib/openoffice.org2.4/share/dict/ooo:/opt/openoffice.org2.3/share/dict/ooo:/usr/lib/openoffice.org2.3/share/dict/ooo:/opt/openoffice.org2.2/share/dict/ooo:/usr/lib/openoffice.org2.2/share/dict/ooo:/opt/openoffice.org2.1/share/dict/ooo:/usr/lib/openoffice.org2.1/share/dict/ooo:/opt/openoffice.org2.0/share/dict/ooo:/usr/lib/openoffice.org2.0/share/dict/ooo
AVAILABLE DICTIONARIES (path is not mandatory for -d option):
/Users/seletz/Library/Spelling/de_CH
/Users/seletz/Library/Spelling/de_AT
/Users/seletz/Library/Spelling/de_AT_small
/Users/seletz/Library/Spelling/de_DE
/Users/seletz/Library/Spelling/de_DE_small
/Users/seletz/Library/Spelling/de_CH_small
LOADED DICTIONARY:
./de_DE.aff
./de_DE.dic
```

## Spell checking in Emacs

To toggle `flyspell mode`, in doom EMACS do `SPC t s` (or `M-x flyspell-mode`).


## links

- [Blog post](https://200ok.ch/posts/2020-08-22_setting_up_spell_checking_with_multiple_dictionaries.html) from Alain M. Lafon about setting up Flyspell in EMACS with multiple dictionaries.
