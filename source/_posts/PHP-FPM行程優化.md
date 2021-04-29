---
title: PHP-FPM行程優化
date: 2021-04-29 17:21:21
tags:
  - PHP-FPM
---

## 前言

近期公司內部 PHP-FPM 有頂到一些行程相關的上限，導致網頁要處理後端程式時無法處理，進而造成服務異常。 而這樣子的問題不外乎可以從 LOG 去看到一些觸發到這個問題的根本原因是什麼。

以下列了兩個行程不夠時常見的 Error Log

```
WARNING: [pool www] server reached pm.max_children setting (5), consider raising it
```

```
WARNING: [pool www] seems busy (you may need to increase pm.start_servers, or pm.min/max_spare_servers), spawning 32 children, there are 0 idle, and 19 total child
```

<!--more-->

PHP-FPM 行程優化相關參數，以下為預設值，需要依照每台機器狀態不同去調教

```
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3

pm.max_requests = 500
pm.process_idle_timeout = 10s # 需搭配 pm = ondemand
```

以下會針對每個項目去解釋其用途

## pm (Process Manager)

pm 為行程管理(Process Manager) 的縮寫，此參數主要是在設定 pm 要使用哪種模式來管理行程，可以區分兩大類型，共三種模式可以選擇，預設是使用 dynamic

### 固定行程數量
- static: 設定此模式，只有 `pm.max_children` 參數會生效，行程數量則根據 `pm.max_children`，效能較好，因為 child process 會保持一個固定的數量，但是比較佔記憶體，因為即便請求較少量時，依然會佔用，所以較不建議使用此模式

### 動態行程數量

皆根據 `pm.max_children`、`pm.start_servers`、`pm.min_spare_servers`、`pm.max_spare_servers` 動態調整

- dynamic: 根據使用量用多寡開行程，但當使用量比較低時，會保留固定行程(根據 `pm.min_spare_servers` 或 `pm.max_spare_servers`)，隨時等著接收新的連線
- ondemand: 用多少量就開多少行程

#### dynamic 實際運作

當我們設定 `pm = dynamic` 並啟動時，首先會產生一定數量的行程(根據 `pm.start_servers`)，此參數可以想像成是最小數量的子程序，而最大數量的子程序則根據 `pm.max_children`，有了最大和最小子程序數量，也就是說服務過程中子程序數量會在最大和最小子程序數量間變化。

而閒置的子程序數量則由 `pm.min_spare_servers`、`pm.max_spare_servers` 控制，超過 `pm.max_spare_servers` 的閒置子程序則會被 Kill。

因為 dynamic 模式可以針對伺服器的回應做最大的優化，但相反的代價是可能造成更多記憶體的佔用，這邊舉例來說，假設 `pm.max_spare_servers = 10`、`pm.max_children = 20`，在某個尖峰時段，這時候最大數量的子程序 20 個都處於忙碌狀態，0個閒置的子程序，過了尖峰時段，請求下降，而當初忙碌的 20 個子程序目前處於閒置狀態，此時 PHP-FPM 只會 Kill 掉 10 個子程序，而最後剩下 10 個閒置的子程序來等待請求，這也就是會造成在尖峰後請求數大量減少後，記憶體卻沒有大量降低的主要原因，而如果把主機重啟，記憶體則會將得更低，這是因為重啟用子程序數量會變成最小閒置行程數量(`pm.min_spare_servers`)

#### ondemand 實際運作

`pm = ondemand` 的運作方式與 dynamic 恰好相反，dynamic 針對伺服器的回應做最大的優化，而 ondemand 則是最佳化記憶體。

而運作方式是，每個閒置的子程序在持續閒置超過 `pm.process_idle_timeout` 設定的時間，就會被 Kill 掉，這樣的模式讓主機在尖峰時期過後可以自然地降低記憶體的使用率，如果長時間都沒有請求連線時，只會有一個 PHP-FPM 主程序。

那壞處則是，如果遇到尖峰時期，或 `pm.process_idle_timeout` 設定得太短的話，會造成伺服器頻繁的建立子程序，而造成效能不佳

## pm.max_children(最大行程數量)

`pm.max_children` 參數在 `pm` 等於 `static` 即為開啟的行程數，而在 `dynamic` 與 `ondemand` 模式下，則是代表最大可開啟的行程數量。

最大行程數量的多寡是根據本身主機的記憶體大小來決定，當記憶體越大時，就可以設定較多的行程數量。 此設定值如果設太小的話會造成處理請求的速度下降，設定太大的話會造成當機，因為記憶體耗盡，所以要依硬體資源來調較。

可以利用以下指令去查詢行程使用的記憶體量

```
$ ps -ylC php-fpm --sort:rss

S   UID     PID    PPID  C PRI  NI   RSS    SZ WCHAN  TTY          TIME CMD
S     0       1       0  0  80   0  4760 20195 do_epo ?        00:02:11 php-fpm
S    33       9       1  0  80   0 20260 22581 -      ?        00:01:20 php-fpm
S    33       8       1  0  80   0 20464 22582 -      ?        00:01:42 php-fpm
S    33       7       1  0  80   0 20476 22584 -      ?        00:01:43 php-fpm
```

RSS 欄位為子程序使用的記憶體大小，單位為KB，20476 約莫 20MB，但是因為每個子程序所使用的記憶體大小都不太一樣，所以我們可以使用下面的指令來求出子程序平均使用記憶體大小

```
ps --no-headers -o "rss,cmd" -C php-fpm | awk '{ sum+=$1 } END { printf ("%d%s\n", sum/NR/1024,"Mb") }'
```

至於 `pm.max_children` 應該要設定多少才合理，以下有一個計算公式

```
pm.max_children = Total RAM / Max child process size
```

> 這邊的 Total RAM 需要扣除已經使用的記憶體，例如系統或其他程序使用後所剩餘的記憶體

## pm.start_servers(初始行程數量)

`pm.start_servers` 可設定 PHP-FPM 服務在一開始啟動時，要配置多少個行程

`pm.start_servers` 同樣有一個計算公式可以參考

```
pm.start_servers = min_spare_servers + (max_spare_servers - min_spare_servers) / 2
```

## pm.min_spare_servers(最小閒置行程數量)

`pm.min_spare_servers` 設定 PHP-FPM 最小閒置行程的數量

## pm.max_spare_servers(最大閒置行程數量)

`pm.max_spare_servers` 設定 PHP-FPM 最大閒置行程的數量

> 這邊需要注意的是 `pm.max_spare_servers` 的值只能小於等於 `pm.max_children`

## pm.max_requests(單一行程可處理連線數)

`pm.max_requests` 可設定單一 PHP-FPM 最多可以處理多少個連線，當一個行程處理的連線數達到設定值時，此行程會被 Kill 掉，而重新產生另一個新的行程

## 參考資料

- https://kejyuntw.gitbooks.io/ubuntu-learning-notes/content/web/php/web-php-max-children.html
- https://support.plesk.com/hc/en-us/articles/360011988053-How-to-calculate-pm-max-children-value-
- https://www.mdeditor.tw/pl/p9Vf/zh-tw
