---
title: hexo-filter-nofollow
tags:
  - Hexo
  - blog
categories:
  - Hexo
---

## 前言

hexo-filter-nofollow 套件可以讓搜尋引擎的 sipder 來爬取站點內容時，不要因為有外部連結的內容，而導致 spider 直接跑到外部連結去，這有可能導致 sipder 不會再回頭了。

hexo-filter-nofollow 可以自動幫我們在外部連結加上 `rel="noopener external nofollow noreferrer"` 的屬性，以便防止這樣的遺憾發生。

<!-- more -->

## hexo-filter-nofollow 的使用

### 安裝與確認

安裝指令

```bash
npm install hexo-filter-nofollow --save
```

確認是否有正確安裝

```bash
npm list | grep "hexo-filter-nofollow"
```

也可以直接到 node_module 目錄底下去看有沒有 hexo-abbrlink 套件

```bash
cd ./node_modules
ls | grep "hexo-filter-nofollow"
```

### 設定檔調整

編輯設定檔

```bash
vim ./_config.yml
```

新增以下片段
```
nofollow:
  enable: true
  field: site
  exclude:
    - 'exclude1.com'
    - 'exclude2.com'
```

選項說明就直接引用 [GitHub](https://github.com/hexojs/hexo-filter-nofollow) 上的說明，如果有要排除的域名，可以寫在 exclude，如果沒用到就註解掉

- enable - Enable the plugin. Default value is `true`.
- field - The scope you want the plugin to proceed, can be 'site' or 'post'. Default value is site.
  - 'post' - Only add nofollow attribute to external links in your post content
  - 'site' - Add nofollow attribute to external links of whole sites
- exclude - Exclude hostname. Specify subdomain when applicable, including www.
  - 'exclude1.com' does not apply to www.exclude1.com nor en.exclude1.com.

## 參考資料

- https://github.com/hexojs/hexo-filter-nofollow