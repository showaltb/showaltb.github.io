---
title: "How to Show Git Status in Your Bash Prompt"
layout: post
---

Here is how to have your bash prompt tell you if you have changed or untracked
files in your working directory. The key is to set a couple of environment
variables that are used by the `__git_ps1` function that `git` provides.

Add these lines to `~/.bash_profile`:

```bash
source /usr/local/etc/bash_completion
GIT_PS1_SHOWDIRTYSTATE=true
GIT_PS1_SHOWUNTRACKEDFILES=true
export PS1='\u@\h \w$(__git_ps1) \$ '
```

(The location for `bash_completion` will vary; this is for OSX with homebrew
installed git. The actual file you need is `git-prompt.sh`, which is
distributed as part of git, but is copied by the installer to the
`bash_completion.d` directory that `bash_completion` loads.)

The prompt for a clean working directory shows the current branch in
parentheses:

```
showaltb@Bobs-MBP ~/work/myproject (develop) $
```

A `%` character indicates untracked file(s). For example, let's create a new
file:

```
showaltb@Bobs-MBP ~/work/myproject (develop) $ touch foo
showaltb@Bobs-MBP ~/work/myproject (develop %) $ git status
On branch develop
Your branch is up-to-date with 'origin/develop'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

        foo

nothing added to commit but untracked files present (use "git add" to track)
```

A `*` character indicates changed or deleted file(s). Let's try removing a
file.

```
showaltb@Bobs-MBP ~/work/myproject (develop) $ rm README.md 
showaltb@Bobs-MBP ~/work/myproject (develop *) $ git status
On branch develop
Your branch is up-to-date with 'origin/develop'.
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        deleted:    README.md

no changes added to commit (use "git add" and/or "git commit -a")
```
