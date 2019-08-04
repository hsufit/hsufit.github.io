---
title: Hexo Create Your Content.md
date: 2019-08-04 21:48:41
tags:
---
## Hexo Writing
### Make new post
Generate page using following command:
```bash
$ hexo new [<layout type>] <post title> 
```
Edit the page content by your editor, such as
```bash
$ vim ./source/_post/<your post name>
```
Then generate the page by hexo commands.
After that, you can preview it on browser(http:localhost:4000).
Or just deploy it to your enviroment.

## Hexo Layout
### What is Fallback mechanism
If the layout is not exist, it use the Fallback layout.
It's not about things below at beginning of your post.
```
---
title: hexo-generate-your-content.md
date: 2019-08-04 21:48:41
tags:
---

```

