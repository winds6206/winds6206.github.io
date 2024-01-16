---
title: Auto Completion不區分大小寫
tags:
  - Bash
  - Auto Completion
  - Bash Completion
  - Linux
categories:
  - Linux
abbrlink: 3bc3c62d
date: 2021-05-11 16:25:56
---

## 前言

Bash Completion 可以讓你在使用 Bash 指令時，按「Tab」鍵，讓系統自動幫你把指令補齊，這可以加速你在輸入指令時的速度。

但如果今天我想要進入 Test 的目錄，依照習慣預設輸入法會是小寫所以大部的人會輸入成

- 輸入 `cd t` 再按 Tab 鍵

此時系統就無法幫忙補齊，因為預設會區分大小寫，如果要忽略大小寫可以參考以下方式

<!--more-->

## 解決方式

在家目錄底下有個 `.inputrc` 的檔案，可以最後面補上 `set completion-ignore-case On` 即可。 或者直接輸入以下指令

```
$ echo 'set completion-ignore-case On' >> ~/.inputrc
```

## 參考資料

- https://caloskao.org/linux-make-tab-auto-completion-case-insensitive-in-the-terminal/
