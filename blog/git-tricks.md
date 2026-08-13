---
title: git tricks
draft: true
date: 2025-02-07
description: A collection of git tricks that I've found useful.
tags: tooling
---

Here seems to be an easy solution found here. Adding the file / directory to .gitignore only is not enough, but git will allow you to manually "ignore" changes to a file / directory:

git update-index --assume-unchanged <file>
And if you want to start tracking changes again, you can undo the previous command using:

git update-index --no-assume-unchanged <file>
To view files for which change tracking is disabled, you can use (linux/unix):

git ls-files -v | grep ^[h]
or (windows):

git ls-files -v | find "h "
Background: I needed this to add a configuration file with user data to the repo. The user should pull the file and edit it for his/her system, but the file should subsequently be ignored by git and never commited.
