---
title: Mac homebrew-core/homebrew-cask is a shallow clone
tags:
  - Mac
  - homebrew
  - brew
categories:
  - Other
abbrlink: e7b1d5c2
---

## 前言

今天剛好要在 Mac 試著安裝 K3s 來試試，Mac 的安裝方式是使用 Homebrew 的方式來安裝，但是當我安裝到最後出現一些錯誤訊息，接著我拜訪 Google大神，大神告訴我要進行 `brew clean && brew update`

但是當我輸入 `brew update` 就無情的跳出以下錯誤訊息

<!-- more -->

```
Error:
 homebrew-core is a shallow clone.
 homebrew-cask is a shallow clone.
To `brew update`, first run:
 git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core fetch --unshallow
 git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask fetch --unshallow
These commands may take a few minutes to run due to the large size of the repositories.
This restriction has been made on GitHub's request because updating shallow
clones is an extremely expensive operation due to the tree layout and traffic of
Homebrew/homebrew-core and Homebrew/homebrew-cask. We don't do this for you
automatically to avoid repeatedly performing an expensive unshallow operation in
CI systems (which should instead be fixed to not use shallow clones). Sorry for
the inconvenience!
```

## 解決方式

我根據上述的指令「分別」運行了以下兩行指令

```
git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core fetch --unshallow
git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask fetch --unshallow
```

當我運行第一行時，大概過了一分鐘，電腦都沒有動靜，原本以為是不是當機了，所幸我就繼續等下去約莫 5~10分鐘後，終端機就開始有反應了，所以這邊要注意的是，當指令輸入後，需要一段時間的耐心等待，指令運行結果如下：

```
$ git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core fetch --unshallow

remote: Enumerating objects: 783435, done.
remote: Counting objects: 100% (783382/783382), done.
remote: Compressing objects: 100% (261076/261076), done.
remote: Total 773593 (delta 521667), reused 761412 (delta 509643), pack-reused 0
Receiving objects: 100% (773593/773593), 309.85 MiB | 5.96 MiB/s, done.
Resolving deltas: 100% (521667/521667), completed with 8031 local objects.
From https://github.com/Homebrew/homebrew-core
  8c9395d92b..37e36e1536  master     -> origin/master
```

```
$ git -C /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask fetch --unshallow

remote: Enumerating objects: 440238, done.
remote: Counting objects: 100% (440214/440214), done.
remote: Compressing objects: 100% (130363/130363), done.
remote: Total 433313 (delta 310180), reused 425457 (delta 302375), pack-reused 0
Receiving objects: 100% (433313/433313), 189.23 MiB | 8.34 MiB/s, done.
Resolving deltas: 100% (310180/310180), completed with 5087 local objects.
From https://github.com/Homebrew/homebrew-cask
  8a81ad4b60..76273d1300  master     -> origin/master
```

最後再進行一次 `brew update` 就正常可以執行了，雖然我不是很清楚實際原因，但大概跟 Homebrew 抓源有一點關係，總之問題就這樣解決了，如果有遇到的朋友們，也可以參照試試看
