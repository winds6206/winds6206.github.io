---
title: Hexo的標籤功能
tags:
  - Hexo
  - blog
categories:
  - Hexo
date: 2021-07-21 11:32:57
---

## 前言

Hexo 網站中的文章可以使用標籤(tags)，標籤有點像是關鍵字的作用，可以替每篇不同的文章給予不同的標籤，這可以讓讀者針對標籤來快速篩選文章

<!--more-->

## 標籤的啟用/使用

啟用標籤功能，讓標籤連結出現在網站上，首先調整 `./themes/next/_config.yml` 中的下述設定，將 `tags: /tags/ || fa fa-tags` 註解「取消」，這樣就可以讓網站出現標籤的連結

```
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
  #schedule: /schedule/ || fa fa-calendar
  #sitemap: /sitemap.xml || fa fa-sitemap
  #commonweal: /404/ || fa fa-heartbeat
```

只有連結並沒有用處，點擊標籤連結時，要能夠顯示相關頁面，所以使用下述指令，讓 Hexo 幫你產生連結檔

```
$ hexo new page tags
```

Hexo 會在 `./source` 底下產生 tags 目錄，目錄內會有一個 index.md 檔案

tags 頁面建立後，還需將 index.md 檔案打開，並且加入 type 參數，如下

```
---
title: tags
date: 2021-04-20 15:37:19
type: "tags"
---
```

這樣基本上標籤功能就已經完成設定

在使用時，只要在文章建立後加上 tags，並把要標籤的值加上去即可，如下

```
---
title: 標籤測試
tags:
  - TEST
  - TAG
date: 2021-05-22 16:17:39
---
```

## 後記

該篇與 Hexo的分類/關於功能 設定方式極度雷同，如有需要可以搭配一起設定，另外兩篇請參考下面連結

- [Hexo的分類功能](https://blog.tonyjhang.tk/2021/07/23/Hexo%E7%9A%84%E5%88%86%E9%A1%9E%E5%8A%9F%E8%83%BD/)
- [Hexo的關於功能](https://blog.tonyjhang.tk/2021/07/26/Hexo%E7%9A%84%E9%97%9C%E6%96%BC%E5%8A%9F%E8%83%BD/)
