---
title: Hexo建立草稿
tags:
  - Hexo
categories:
  - Blog
abbrlink: 946e0e0b
date: 2021-05-03 10:26:06
---

## 前言

使用 Hexo 來當作技術部落格不僅簡單且快速，在剛開始架設完成後，可以使用指令 `hexo new [title]` 來建立文章並且撰寫，但是這樣的方式會延伸一個問題，如果這篇文章還處理撰寫的階段，但是我有修改其他已完成的文章需要發佈出去，但是不想把未完成的文章也一起發佈，此時就會有一些困擾，此篇會來說明如果使用 Hexo 的草稿功能。

<!--more-->

## Hexo 草稿

一般來說，我們建立文章都是使用此指令

```bash
hexo new [Title] [FileName]
```

如果今天要建立 Hexo 草稿，我們可以將上述指令改成

```bash
hexo new draft [Title] [FileName]
```

那此時產生出來的草稿會放在 `./source/_drafts`

> 如果是使用 `hexo new [title]` 則會放在 `./source/_posts`

當我們要發佈一篇文章時，我們會使用 `hexo g` 和 `hexo d` 來編譯產生靜態檔與發佈文章，而草稿目錄內的文章不會被發佈出去，是 **因為 `hexo g` 並不會編譯 `./source/_drafts` 底下的檔案**。

如果放在草稿目錄底下的文章完成後，想在本機先預覽檢查時，可以使用此指令

> 此處「文後討論」有補充說明

```bash
hexo s --draft
```

確認文章沒問題後，我們就可以將草稿 publish，使用此指令

> 這邊的 publish 是將檔案從 _drafts 移動到 _posts

```bash
hexo publish [FileName]  # 這邊是接 FileName，不包含副檔名 .md
```

接下來就跟放在 _posts 底下的文章發佈沒有兩樣，先進行移除舊的靜態檔與快取

```bash
hexo cl
```

然後進行編譯的動作

```bash
hexo g
```

最後再發佈

```bash
hexo d
```

## 文後討論

看完上述，你可能會想，那能不能讓 `hexo s` 自動將 _drafts 底下的檔案自動預覽出來，而不要在指令後面又多加一個參數 `--draft`?

答案是 可以的!

只要去調整 `./_config.yml` 內的設定，將下面的參數值由 false 調整為 true 即可

> 非 themes 底下的 _config.yml

```yaml
render_drafts: true
```

調整之後，以後要預覽 _drafts 內的文章，就跟一般放在 _posts 底下的一樣，只要使用 `hexo s` 通通都可以在本機預覽得到。

## 參考資料

- https://www.dazhuanlan.com/2019/11/02/5dbc7ecc737a5/
