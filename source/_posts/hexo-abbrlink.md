---
title: 使用 hexo-abbrlink 優化文章連結
tags:
  - Hexo
categories:
  - Blog
abbrlink: 94411fbc
date: 2024-01-24 16:38:31
---


## 前言

Hexo 產生文章連結時，預設是使用「年/月/日/文章Title」的格式，導致 URL 太多層，如果是中文標題，複製下來的連結會過於冗長，這不僅不利於 SEO，也容易在二次編輯文章時，導致原先的連結失效。

最好的方式就是將文章的 URL 固定下來，這樣就可以一勞永逸，今天會使用 hexo-abbrlink 的套件來幫助我們達到固定 URL，而且不重複。

<!-- more -->

## hexo-abbrlink 的使用

這邊先提供 hexo-abbrlink 的 github 專案連結：

- https://github.com/rozbo/hexo-abbrlink

以下的設定是依據我自身的設定提供參考

### 安裝與確認

安裝指令

```bash
npm install hexo-abbrlink --save
```

確認是否有正確安裝

```bash
npm list | grep "hexo-abbrlink"
```

也可以直接到 node_module 目錄底下去看有沒有 hexo-abbrlink 套件

```bash
cd ./node_modules
ls | grep "hexo-abbrlink"
```

### 設定檔調整

編輯設定檔

```bash
vim ./_config.yml
```

新增以下片段，這邊只提供自身的設定，如果有需要更 detail 的設定可以參照 [hexo-abbrlink](https://github.com/rozbo/hexo-abbrlink) GitHub 專案

```yaml
abbrlink:
  alg: crc32      #support crc16(default) and crc32
  rep: hex        #support dec(default) and hex
  drafts: true   #(true)Process draft,(false)Do not process draft. false(default)
```

註解預設的 permalink，並且在下面增加新的設定，之後就會依照該格式產生連結
```yaml
#permalink: :year/:month/:day/:title/
permalink: posts/:abbrlink/
```

### 使用方式

上述步驟都設定好之後，重新執行下述指令

```bash
hexo clean
hexo generate
```

套件就會依據 Title 自動產生一組亂數的 abbrlink 在文章的 markdown 原檔案上方，如下

```text
---
title: hexo-abbrlink 測試
tags:
  - Hexo
categories:
  - Hexo
abbrlink: 11e41c6
---
```

abbrlink 一旦產生後，不論 Title 怎麼修改，abbrlink 都不會變動，除非將 abbrlink 刪除，重新 `hexo generate`

確定 abbrlink 正常產出後，就可以 `hexo server` 進行本機測試，此時可以發現 URL 已經依照上面的設定改變了，會變成下面這種格式

```text
http://www.example.com/posts/11e41c6
```

最後確認無誤就可以佈署發布囉

```bash
hexo deploy
```

## 參考資料

- https://github.com/rozbo/hexo-abbrlink