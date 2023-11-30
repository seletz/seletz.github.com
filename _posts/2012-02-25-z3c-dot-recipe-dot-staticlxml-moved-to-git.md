---
layout: post
title: "z3c.recipe.staticlxml moved to git"
date: 2012-02-25 14:35
comments: true
tags: tools python
---

I've got a request to update my buildout recipe to build a statically
linked lxml library, I decided to move the repository to GitHub.  This way,
people can fork my code and submit pull requests.

As I ran into some issues during the process I did a quick writeup.

<!-- more -->

Moving to git
-------------

First I cheked GitHub for a [tutorial] [1] -- I'm lazy.

Then I tried to launch `git-svn` in my shell, tried to install it, until
I remembered that the actual command is `git svn`.  D'oh.

Then I ran into this error when importing the SVN repo:

	using existing [svn-remote "svn"]
	Using higher level of URL: svn+ssh://seletz@svn.zope.org/repos/main/z3c.recipe.staticlxml => svn+ssh://seletz@svn.zope.org/repos/main
	W: Ignoring error from SVN, path probably does not exist: (160013): Filesystem has no item: File not found: revision 100, path '/z3c.recipe.staticlxml'
	W: Do not be alarmed at the above message git-svn is just searching aggressively for old history.
	This may take a while on large repositories
	r93394 = b27bb31b8e3f9048d7fbf639ebd89e6b33f47f36 (refs/remotes/trunk)
		A	bootstrap.py
		A	buildout.cfg
		A	CHANGES.txt
		A	setup.py
		A	src/z3c/__init__.py
		A	src/z3c/recipe/staticlxml/tests/__init__.py
		A	src/z3c/recipe/staticlxml/tests/test_docs.py
		A	src/z3c/recipe/staticlxml/__init__.py
		A	src/z3c/recipe/staticlxml/README.txt
		A	src/z3c/recipe/__init__.py
		A	CONTRIBUTORS.txt
		A	README.txt
	W: +empty_dir: z3c.recipe.staticlxml/trunk/downloads
	r93395 = 6898a232ff85dbfd6aff4e4f9dc959fe3b9fd443 (refs/remotes/trunk)
		M	buildout.cfg
		M	CHANGES.txt
		M	src/z3c/recipe/staticlxml/__init__.py
		M	src/z3c/recipe/staticlxml/README.txt
	r93396 = 40feb0aa5f65476cf6fa7b8722e164660dd0c4e2 (refs/remotes/trunk)
		M	buildout.cfg
		M	CHANGES.txt
		M	src/z3c/recipe/staticlxml/__init__.py
	r93397 = 1155246997d81732cdcb466c0b692edcd260d231 (refs/remotes/trunk)
		M	CHANGES.txt]
		M	setup.py
	r93400 = 32375fd9ab8024c0f8b8ed0f40a3dad27a520a25 (refs/remotes/trunk)
	Found possible branch point: svn+ssh://seletz@svn.zope.org/repos/main/z3c.recipe.staticlxml/trunk => svn+ssh://seletz@svn.zope.org/repos/main/z3c.recipe.staticlxml/branches/0.1, 93401
	Use of uninitialized value $u in substitution (s///) at /usr/local/Cellar/git/1.7.9.1/libexec/git-core/git-svn line 2097.
	Use of uninitialized value $u in concatenation (.) or string at /usr/local/Cellar/git/1.7.9.1/libexec/git-core/git-svn line 2097.
	refs/remotes/svn/tags/0.1: 'svn+ssh://svn.zope.org/repos/main' not found in ''

Google reveals that this seems to be a git [issue] [2].

So, I updated to `git 1.7.9.2`, but still the same error.  More googling, and some perl hacking
I ended up with this diff:

	--- /usr/local/Cellar/git/1.7.9.2/libexec/git-core/git-svn.orig	2012-02-25 14:52:46.000000000 +0100
	+++ /usr/local/Cellar/git/1.7.9.2/libexec/git-core/git-svn	2012-02-25 14:53:46.000000000 +0100
	@@ -2094,8 +2094,12 @@
					    " globbed: $refname\n";
				}
				my $u = (::cmt_metadata("$refname"))[0];
	-			$u =~ s!^\Q$url\E(/|$)!! or die
	-			  "$refname: '$url' not found in '$u'\n";
	+			if (!$u) {
	+				$u = $pathname;
	+			} else {
	+				$u =~ s!^\Q$url\E(/|$)!! or die
	+				  "$refname: '$url' not found in '$u'\n";
	+			}
				if ($pathname ne $u) {
					warn "W: Refspec glob conflict ",
					     "(ref: $refname):\n",

This seems to do the trick.

Actual Commandline To Do The Conversion
---------------------------------------

After hacking around the issues I got, the actual process is simple:

	$ mkdir z3c.recipe.staticlxml.git
	$ cd z3c.recipe.staticlxml.git
	$ sudo gem install svn2git
	$ svn2git --verbose svn+ssh://seletz@svn.zope.org/repos/main/z3c.recipe.staticlxml

Then simply add the new remote and push:

	$ git remote add origin git@github.com:seletz/z3c.recipe.staticlxml.git
	$ git push origin master

Now create a development branch:

	$ git co -b develop
	$ git push origin develop

Now I'm ready to hack on this project.

[1]: http://help.github.com/import-from-subversion "tutorial"
[2]: http://help.github.com/import-from-subversion "issue"
