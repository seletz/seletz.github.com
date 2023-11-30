---
layout: post
title: "switch from bash to zsh"
date: 2012-01-06 13:03
comments: true
categories: tools howto
published: true
---

I finally came around to switch from `bash` to `zsh`.  While reading [hacker news] [1]
I found a nice little setup on *github*: [oh-my-zsh] [2]


<!-- more -->

Installation
------------

Installation was quite easy:

```bash
	$ brew install zsh
	$ cd $HOME
	$ git clone https://github.com/robbyrussell/oh-my-zsh.git .oh-my-zsh
	$ sudo chsh -s /usr/local/bin/zsh yourusernamehere
```

**NOTE:**
  You may need to edit `/etc/shells` to have `/usr/local/bin/zsh`.

Configuration
-------------

I did very little configuration.  All I really did is to choose a nice
theme from the many available themes.  Anyhow, my `.zshrc` looks like this:

```bash
# Path to your oh-my-zsh configuration.
ZSH=$HOME/.oh-my-zsh

# Set name of the theme to load.
# Look in ~/.oh-my-zsh/themes/
# Optionally, if you set this to "random", it'll load a random theme each
# time that oh-my-zsh is loaded.
ZSH_THEME="nanotech"

# Example aliases
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"

# Set to this to use case-sensitive completion
# CASE_SENSITIVE="true"

# Comment this out to disable weekly auto-update checks
# DISABLE_AUTO_UPDATE="true"

# Uncomment following line if you want to disable colors in ls
# DISABLE_LS_COLORS="true"

# Uncomment following line if you want to disable autosetting terminal title.
# DISABLE_AUTO_TITLE="true"

# Uncomment following line if you want red dots to be displayed while waiting for completion
# COMPLETION_WAITING_DOTS="true"

# Which plugins would you like to load? (plugins can be found in ~/.oh-my-zsh/plugins/*)
# Custom plugins may be added to ~/.oh-my-zsh/custom/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
plugins=(osx git git-flow fabric groovy grails python)

source $ZSH/oh-my-zsh.sh

# Customize to your needs...

#---------------------------------------------------------------------
# prompt and path
export PATH=~/bin:/usr/local/bin:/usr/local/sbin:/usr/local/mysql/bin:$PATH

#---------------------------------------------------------------------
# env setup
export LANG="de_DE.UTF-8"
# gnuchlog vim plugin
export EMAIL="Stefan Eletzhofer <stefan.eletzhofer@nexiles.de>"
export EDITOR="/Applications/MacVim.app/Contents/MacOS/Vim -g -f "

#---------------------------------------------------------------------
# python
export PYTHONSTARTUP=~/.pyinit

export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python2.6
export WORKON_HOME=~/.virtualenvs
export PROJECT_HOME=$HOME/develop
export VIRTUALENV_ROOT=$WORKON_HOME

export JYTHON_HOME=$(brew --prefix jython)/libexec
export PATH=$PATH:$JYTHON_HOME/bin

. /usr/local/bin/virtualenvwrapper.sh

#---------------------------------------------------------------------
# NODE.JS
export NODE_PATH=/usr/local/lib/node
export JS_CMD=node

#---------------------------------------------------------------------
# Ruby RVM
source ~/.rvm/scripts/rvm

#---------------------------------------------------------------------
# GROOVY
export GROOVY_HOME=$(brew --prefix groovy)/libexec

#---------------------------------------------------------------------
# alias
alias ls="gls --color"
alias la="ls -la"
alias ll="ls -l"
alias serve="python -mSimpleHTTPServer"

#---------------------------------------------------------------------
# direnv hook
eval `direnv hook $0`
```


  [1]: https://news.ycombinator.com                   "hackernews"
  [2]: https://github.com/robbyrussell/oh-my-zsh      "oh-my-zsh"
  [3]: http://ethanschoonover.com/solarized           "solarized"

