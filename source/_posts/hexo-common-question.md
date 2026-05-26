---
title: Hexo Common Question
date: 2019-08-04 16:05:39
pubDatetime: 2019-08-04T16:05:39+08:00
modDatetime: 2019-08-09T15:29:58+08:00
description: Common Hexo notes for deployment, installing the git deployer, and deleting posts.
tags:
- hexo
---

## About Deployment
### How can i generate and deploy at one command
Use following command:
```bash
$ hexo deploy -g
```

###What should I do if `Deployer not found: git` error message occureed
Install deployment tool with this command:
```bash
npm install hexo-deployer-git --save
```

## Misc
### How to delete a post
run:
```
rm ./source/_post/<post you want to delete>
hexo clean
```
then regenerate the page 
