---
title: Hexo的關於功能
tags:
  - Hexo
categories:
  - Blog
abbrlink: 8be085a8
date: 2021-07-26 10:26:48
---

## 前言

Hexo 的關於(about) 功能，主要是可以讓讀者點進特定頁面來去了解作者本人，也就是作者的自我介紹頁，以下會介紹如何啟用與使用

<!--more-->

## 關於的使用

啟用關於功能，讓關於連結出現在網站上，首先調整 `./themes/next/_config.yml` 中的下述設定，將 `about: /about/ || fa fa-user` 註解「取消」，這樣就可以讓網站出現關於的連結

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

只有連結並沒有用處，點擊關於連結時，要能夠顯示相關頁面，所以使用下述指令，讓 Hexo 幫你產生連結檔

```bash
hexo new page about
```

Hexo 會在 `./source` 底下產生 about 目錄，目錄內會有一個 index.md 檔案

about 頁面建立後，還需將 index.md 檔案打開，並且加入 type 參數，如下

```
---
title: about
date: 2021-04-20 15:37:19
type: "about"
---
```

這樣基本上關於功能就已經完成設定

使用時，只要在 `./source/about/index.md` 頁面以 Markdown 語法寫下自我介紹即可

## 後記

該篇與 Hexo的分類/標籤功能 設定方式極度雷同，如有需要可以搭配一起設定，另外兩篇請參考下面連結

- [Hexo的分類功能](https://blog.tonyjhang.dev/posts/8bfb5405)
- [Hexo的標籤功能](https://blog.tonyjhang.dev/posts/1436c3e2)
