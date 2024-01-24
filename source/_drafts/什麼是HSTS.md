---
title: 什麼是HSTS
tags:
  - HSTS
  - SSL
  - HTTPS
  - Nginx
categories:
  - Nginx
abbrlink: fbf0c319
---

## 前言

HTTP 強制安全傳輸(HTTP Strict Transport Security, HSTS)，就是告訴瀏覽器該網站請強制使用 HTTPS 協定作加密連線，以減少連線過程可能遭遇的風險

開啟 HSTS 後未來瀏覽器就會自動將 HTTP 轉為 HTTPS 請求，**即使憑證失效，使用者也無法忽略警告繼續瀏覽網站**。

在 HTTP Repose Header 加入 Strict-Transport-Security(STS) 參數就能告訴瀏覽器該網站使用 HTTPS 連線，在遇到使用 HTTP 訪問時就能自動跳轉到 HTTPS。

<!--more-->

## HTTP To HTTPS 跳轉過程

在還沒有 HTTPS 的年代，都以 80 port 明碼的方式與伺服器通訊，正常情況下可以返回資料

![](HSTS-0.png)

但如果瀏覽器與伺服器之間出現了 Hacker 劫持，伺服器所返回的資料就有可能被有所竄改

![](HSTS-1.png)

後來 SSL加密的出現，我們可以在伺服器端利用 301/302 跳轉強制讓走 80 的客戶自動導轉 443，這也是為了因應平時我們會在瀏覽器習慣性輸入 `www.example.com`。

當瀏覽器走的是 HTTP，請求到達伺服器後，伺服器告訴瀏覽器 302 跳轉，然後瀏覽器重新請求，透過 HTTPS 方式，443 埠號通訊。

![](HSTS-2.png)

也因為不是直接輸入 `https://`，故第一次發起的請求還是使用 80 port，Hacker 利用這一點，也可以進行劫持的動作

![](HSTS-3.png)

## HSTS 運作原理

1. 伺服器啟用 HSTS 功能，伺服器會在 Repose Header 中新增 Strict-Transport-Security，並設置 max-age 時間

2. 客戶端第一次與伺服器連線時，伺服器回應時會帶上 STS Header

3. 如果下次客戶端使用 HTTP 訪問時，只要 max-age 未過期，客戶端會進行內部 307 跳轉，使用 Chrome 瀏覽器開啟「開發人員工具」，可以看到 307 Redirect Internel 的狀態碼

4. Redirect Internel 是客戶端本機瀏覽器 HTTP To HTTPS 跳轉過程，跳轉完成後客戶端就會以 HTTPS 訪問伺服器，這樣就能確保客戶端訪問伺服器時是使用 HTTPS 加密連線

> 總結：使用 HSTS 時，當 max-age 未過期，客戶端會在「自己的瀏覽器」進行 307 跳轉成 HTTPS，再去訪問伺服器

## HSTS 優點

1. 啟用 HSTS 可以有效降低 MITM

2. 保客戶端在有效時間內使用 HTTPS 訪問伺服器

## HSTS 缺點

1. 在瀏覽器未有 STS Header 時，首次使用 80 port 訪問伺服器時，有可能遭受 MITM

2. 無法忽視憑證過期或失效，只要憑證過期或失效就無法訪問網站

3. 如果客戶端在接收到 Strict-Transport-Security(STS) Response Header 前就已經是被劫持狀態，因客戶端並非直接與伺服器連線，故伺服器無法將 STS Response Header 帶給客戶端(因為都被中間人擋下來了)

4. 若清除瀏覽器快取，下次訪問網站時，會使用 HTTP 訪問，伺服器回應會重新帶上 STS Header

> 關於第一點首次訪問使用 HTTP 的問題，其實可以使用 Preload List 的方式來解決，後續會再寫一篇有關 Preload List

## HSTS 的注意事項

1. IP 的請求 HSTS 無法處理，例如：`http://1.1.1.1` Response Header 中設置了 STS，瀏覽器也不會理會

2. HSTS 只能在 80 和 443 埠號之間的轉換，如果服務是 8080 埠號，即便設置了 STS 也無效

3. 當憑證一旦失效，網站會無法順利開啟

4. 如果伺服器的 HTTPS 沒有設定好就開啟了 STS 的 Response Header，並且設置了很長的過期時間，那麼在你伺服器 HTTPS 設定好之前，用戶都是沒辦法連到伺服器，除非 max-age 過期了(因為客戶端會強制使用 HTTPS 與伺服器連線)

## 參考資料

- https://free.com.tw/hsts-preload-list/
- https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/668827/
