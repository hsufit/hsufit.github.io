---
title: Hexo Quick Start
date: 2020-04-22 22:04:34
tags:
- hexo
---

## Install Node js and hexo
### Install nvm to manage Node.js version
``` bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
```
### Install Node.js
``` bash
nvm install node
```
### Install hexo
``` bash
npm install -g hexo-cli
```
## Install hexo modules in your hexo folder
### For essestial command
``` bash
npm install hexo --save
```
### For server
``` bash
npm install hexo-server
```
### Other dependency:
``` bash
npm install hexo-admin --save
npm install hexo-generator-archive --save
npm install hexo-generator-feed --save
npm install hexo-generator-search --save
npm install hexo-generator-tag --save
npm install hexo-deployer-git --save
npm install hexo-generator-sitemap --save
```
### Or autometically install all dependency
```
npm install
```

## Then join hexo with fun
``` bash
hexo init
```


