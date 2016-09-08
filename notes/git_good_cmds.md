---
title: good git commands
---

# {{ page.title }}

* just show files with git show

    git show --pretty="format:" --name-only $commit

* delete remote branch

    git push $remote :$yourbranch

* delete remote tag

    git push $remote :refs/tags/$yourtag

* fetch remote branch

    git  checkout --track  $remote/$remotebranch

* delete remote tracking branch

    git branch -d -r $remote/$branch

* directly checkout branch with git clone

    git clone --branch $branch $projecturl

* compare one file between two commits

    git diff $commit1..$commit2 $filename

* show the whole historical commits using a tree-liked styles through all branches

    git log --graph --online --decorate --all

* show all authors/emails

    git log --format='%aN' | sort -u
    git log --format='%aE' | sort -u

* show git history for one user from one date and apply with some pattern, then give file changes

    git log --author='xxx' --grep='${pattern}' --since='xxxx-xx-xx' --stat

* Very important: restore operation to save your ass when you did some destruction operation, like **git reset --hard**

    git reflog show

* Calculate one author's code line changes

    git log --author="user" --pretty=tformat: --numstat | gawk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s removed lines: %s total lines: %s\n", add, subs, loc }' -

* show all tags with create date by date order

    git for-each-ref --sort=taggerdate --format '%(refname) %(taggerdate)' refs/tags
