---
title: 如何修改PHP-FPM上傳檔案大小限制
tags:
  - PHP-FPM
  - PHP
categories:
  - PHP-FPM
abbrlink: 452fc7c7
date: 2021-05-22 16:17:39
---

## 前言

網頁上傳檔案屬於後端運作，如果檔案太大可能會造成上傳失敗，如果是採 Nginx + PHP-FPM 的架構，此時需要修改後端(PHP-FPM) `php.ini` 的相關參數設定。

可以分為兩個面向：

- 檔案上傳大小限制
- 腳本執行時間或網路連線時間的長短限制

<!--more-->

相關參數如下：

> 此處的數值為預設值

```text
# 檔案上傳大小限制相關參數
upload_max_filesize = 2M
post_max_size = 8M
memory_limit = 128M

# 腳本執行時間或網路連線時間的限制
max_execution_time = 60
max_input_time = 60
default_socket_timeout = 60
```

## 解決方式

依照自身需求，調整 `php.ini` 的相關參數設定

```text
# 上傳單一檔案大小
upload_max_filesize = 100M

# 使用表單 POST 給 PHP 的大小上限(所有檔案大小加總)
post_max_size = 8M

# 單一 PHP 腳本使用記憶體的上限
memory_limit = 128M
```

這邊需要注意的是，上述參數之間的值有大小原則需要注意

```text
memory_limit > post_max_size > upload_max_filesize
```

到此就可以讓上傳檔案的大小再往上提升，不過接下來你可能會遇到檔案傳送到一半會斷線的情況，這代表傳送的檔案較大需要較多的時間，此時就需要再調整有關腳本執行的時間與網路連線的相關設定。

---

有關 PHP 腳本執行的時間與網路連線時間長短的相關參數：

> 此處的數值為預設值

```text
# PHP 腳本執行的時間上限(秒)，可避免無窮迴圈
max_execution_time = 30

# 每個 PHP 腳本接收資料的時間上限(秒)，如果網路較慢，時間可能需要拉長
max_input_time = 60

# Socket 無回應斷線時間上限
default_socket_timeout = 60
```

> `max_input_time` 數值若為 -1 表示無限制時間

此處可以依照自身的 PHP 腳本與網路環境來調整到符合自身的需求

## 參考資料

- https://blog.gtwang.org/web-development/php-ini-large-file-upload-configuration/
