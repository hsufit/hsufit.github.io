---
title: Hexo Create Your Content
date: 2019-08-04 21:48:41
tags:
- hexo
---
## Hexo Writing
### Make new post
Generate page using following command:
```console
$ hexo new [<layout type>] <post title> 
```
Edit the page content by your editor, such as
```console
$ vim ./source/_post/<your post name>
```
Then generate the page by hexo commands.
After that, you can preview it on browser(http:localhost:4000).
Or just deploy it to your enviroment.

## Hexo Layout
### What is Fallback mechanism
If the layout is not exist, it use the Fallback layout.
It's not about things below at beginning of your post.
```markdown
---
title: hexo-generate-your-content.md
date: 2019-08-04 21:48:41
tags:
---
```

## Hexo Editor
[hexo-admin][]

## Reference
[hexo tutorial: writing][hexo-writing]

[hexo-writing]: https://hexo.io/docs/writing.html "hexo tutorial: writing"
[hexo-admin]: https://github.com/jaredly/hexo-admin "hexo admin editor"
