---
title: Hexo的分類功能
tags:
  - Hexo
  - blog
categories:
  - Hexo
abbrlink: 8bfb5405
date: 2021-07-23 10:26:06
---

## 前言

Hexo 的分類功能其實跟資料夾的功能差不多，作者可以自行將文章做分類，讓讀者在找尋時可以依據分類來快速找到相關的文章，功能上與標籤(tags) 有點大同小異

<!--more-->

## 分類的使用

啟用分類功能，讓分類連結出現在網站上，首先調整 `./themes/next/_config.yml` 中的下述設定，將 `categories: /categories/ || fa fa-th` 註解「取消」，這樣就可以讓網站出現分類的連結

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

只有連結並沒有用處，點擊分類連結時，要能夠顯示相關頁面，所以使用下述指令，讓 Hexo 幫你產生連結檔

```bash
hexo new page categories
```

Hexo 會在 `./source` 底下產生 categories 目錄，目錄內會有一個 index.md 檔案

categories 頁面建立後，還需將 index.md 檔案打開，並且加入 type 參數，如下

```
---
title: categories
date: 2021-04-20 15:37:19
type: "categories"
---
```

這樣基本上分類功能就已經完成設定

在使用時，只要在文章建立後加上 categories，並把要分類的值加上去即可，如下

```
---
title: 分類測試
date: 2021-05-22 16:17:39
categories:
  - Hexo
---
```

這樣該篇文章就會被納入 Hexo 這個分類底下

## 文後討論

如果文章內的 categories 設定如下

```
---
title: 分類測試
date: 2021-05-22 16:17:39
categories:
  - Hexo
  - NexT
---
```

此時該文章將被分類到 Hexo 中的 NexT 底下，如：Hexo/NexT/分類測試

## 後記

該篇與 Hexo的標籤/關於功能 設定方式極度雷同，如有需要可以搭配一起設定，另外兩篇請參考下面連結

- [Hexo的標籤功能](https://winds6206.github.io/posts/1436c3e2)
- [Hexo的關於功能](https://winds6206.github.io/posts/8be085a8)
