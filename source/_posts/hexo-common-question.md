---
title: Hexo Common Question
date: 2019-08-04 16:05:39
tags: hexo
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
