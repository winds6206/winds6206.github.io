---
title: TIME_WAIT狀態如何產生
tags:
---

## 前言

<!--more-->

## 產生原因

只要了解 TCP 連線機制與流程，就可以知道 TIME_WAIT 是如何產生的，以下繼續

### TCP 建立連線 (Open)

通常 TCP 連線建立流程，需要經過三向交握(three-way handshaking) 來建立連線

![]()

1. Server 建立 TCB，開啟監聽連線，進入狀態 LISTENING
2. Client 發出連線要求 SYN，進入狀態 SYN-SENT，等待回應
3. Server 收到 SYN 要求，回應連線傳送 SYN+ACK，並進入狀態 SYN-RCVD (SYN-RECEIVED)
4. Client 收到 SYN+ACK 確認完成連線進入狀態 ESTABLISHED，並送出 ACK
5. Server 收到 ACK 確認連線完成，也進入狀態 ESTABLISHED
6. 雙方開始傳送交換資料

該些名詞與狀態說明：
- CLOSED：連線關閉狀態
- LISTENING：監聽狀態，被動等待連線
- SYN-SENT：主動送出連線要求 SYN，並等待對方回應
- SYN-RCVD：收到連線要求 SYN，送出己方的 SYN+ACK 後，等待對方回應
- ESTABLISHED：確定完成連線，可開始傳輸資料
- TCB：Transmission Control Block，see wiki
- SYN：Synchronous，表示與對方建立連線的同步符號
- ACK：Acknowledgement，表示發送的數據已收到無誤


### TCP 斷開連線 (Close)

TCP 關閉流程如下，比建立連線還要複雜一些，需要經過四次的訊息交換 (four-way handshaking)，要注意的是可以是由 server 發起主動關閉，抑或是 client 發起主動關閉：

上圖是 TCP 連線關閉時的四次交握(four-way handshaking)，通常會由 client 主動發起關閉連線

![]()

Client 準備關閉連線，發出 FIN，進入狀態 FIN-WAIT-1
Server 收到 FIN，發回收到的 ACK，進入狀態 CLOSE-WAIT，並通知 App 準備斷線
Client 收到 ACK，進入狀態 FIN-WAIT-2，等待 server 發出 FIN
Server 確認 App 處理完斷線請求，發出 FIN，並進入狀態 LAST-ACK
Client 收到 FIN，並回傳確認的 ACK，進入狀態 TIME-WAIT，等待時間過後正式關閉連線
Server 收到 ACK，便直接關閉連線

該些名詞與狀態說明：
- ESTABLISHED：連線開啟狀態
- CLOSE-WAIT：等待連線關閉狀態，等待 App 回應
- LAST-ACK：等待連線關閉狀態，等待遠端回應 ACK 後，便關閉連線
- FIN-WAIT-1：等待連線關閉狀態，等待遠端回應 ACK
- FIN-WAIT-2：等待連線關閉狀態，等待遠端回應 FIN
- TIME-WAIT：等待連線關閉狀態，等段一段時候，保證遠端有收到其 ACK 關閉連線 (網路延遲問題)
- CLOSED：連線關閉狀態
- FIN：表示關閉連線的同步符號
- ACK：Acknowledgement，表示發送的數據已收到無誤


TCP 連線關閉時，會有 4 次通訊（四次交握），來確認雙方都停止收發數據了。如上圖，主動關閉方，最後發送 ACK 時，會進入 TIME_WAIT 狀態，要等 2MSL 時間後，這條連接才真正消失。

## 文後討論

## 參考資料

- https://dev.twsiyuan.com/2017/09/tcp-states.html
