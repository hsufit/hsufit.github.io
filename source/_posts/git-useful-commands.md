---
title: Git Useful Commands
date: 2020-05-20 21:56:05
tags:
  - git
  - tool
---

## List useful logs
### List log and modified file with path only
```
$ git log --name-only
```
### List log and modified file with path and modified status
```
$ git log --name-status
```
### List log and modified file with path and modified line summary
```
$ git log --stat
```
### list history of one file
```
$ git log --full-history  -- myfile
```

## Find some pattern in git
### Find a deleted file
```
$ git log --diff-filter=D --summary | grep delete
```

### Find a commit message
```
$ git log --grep=word
```

### Find a new or deleted modifycation
```
$ git log -S<string>
$ git log -G<regular expression>
```

reference:
https://stackoverflow.com/questions/1337320/how-to-grep-git-commit-diffs-or-contents-for-a-certain-word
https://stackoverflow.com/questions/7203515/how-to-find-a-deleted-file-in-the-project-commit-history
https://stackoverflow.com/questions/1528513/git-find-deleted-code
https://stackoverflow.com/questions/6839398/find-when-a-file-was-deleted-in-git
https://stackoverflow.com/questions/1230084/how-to-have-git-log-show-filenames-like-svn-log-v/1230094



