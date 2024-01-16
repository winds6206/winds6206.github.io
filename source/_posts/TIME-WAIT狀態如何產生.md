---
title: TIME_WAIT狀態如何產生
tags:
  - TCP
categories: TCP/IP
abbrlink: ab6fcbda
date: 2021-08-17 15:29:31
---


## 前言

之前 Maintain A團隊的 K8s 時，有發生過 golang 寫的服務太多的 TIME_WAIT 導致 Pod 無法建立新的連線。 近期 Maintain B團隊的服務時，因為線上人數變多，擔心 Nginx 向 Upstream 發請求時，會不會造成太多的連線，然後有過多的 TIME_WAIT 現象，因為預設 Nginx 向 Upstream 發送請求是使用 http/1.0，會導致連線無法複用，所以當連線一多時，就很容易發生過多的 TIME_WAIT 狀態。

所以本篇主要是在講解 TCP 協定在「建立連線」與「關閉連線」的過程，只要了解運作過程，就會知道 TIME_WAIT 狀態到底是如何產生的

<!--more-->

## TCP 建立連線

依據 IETF 的標準文件 [rfc793](http://www.rfc-editor.org/rfc/rfc793.txt) 中所描述的情形，可分為以下四種不同狀況

- Basic 3-Way Handshake for Connection Synchronization
- Simultaneous Connection Synchronization
- Recovery from Old Duplicate SYN
- Half-Open Connection Discovery

以下僅針對第一個 Basic 3-Way Handshake for Connection Synchronization 進行說明

通常 TCP 連線建立流程，需要經過三向交握(three-way handshaking) 來建立連線

![](TCPIP-0.png)

1. Server 建立 TCB，開啟監聽連線，進入 LISTENING 狀態
2. Client 主動發出連線請求 SYN，進入 SYN_SENT 狀態，並等待回應
3. Server 收到 SYN 要求，回應連線 SYN,ACK，並進入 SYN_RCVD 狀態
4. Client 收到 SYN,ACK 確認完成連線進入 ESTABLISHED 狀態，並送出 ACK
5. Server 收到 ACK 確認連線完成，同時也進入 ESTABLISHED 狀態

名詞解釋：
- CLOSED：已關閉連線，表示該主機的連線呈現關閉中
- LISTENING：監聽狀態，表示該主機被動等待連線請求
- SYN_SENT：表示已送出 SYN 訊息，並等待對方回應
- SYN_RCVD：表示已接收到對方的 SYN 訊息，並且送出 SYN,ACK，等待對方回應
- ESTABLISHED：表示已完成雙方連線的建立，可開始傳輸資料
- TCB：傳輸控制區塊(Transmission Control Block)，用來儲存 Server 端有關 TCP 的所有資訊
- SYN：Synchronous，表示與對方建立連線的請求
- ACK：Acknowledgement，表示發送的數據已收到無誤

「TCP建立連線」狀態流程圖

![](TCPIP-1.png)

## TCP 關閉連線

依據 IETF 的標準文件 [rfc793](http://www.rfc-editor.org/rfc/rfc793.txt) 中所描述的情形，可分為以下二種不同狀況

- Normal Close Sequence
- Simultaneous Close Sequence

以下僅針對第一個 Normal Close Sequence 進行說明

TCP 關閉流程如下，需要經過四次交握 (four-way handshaking)，來確認雙方都停止收發數據，**要注意的是可以是由 server 發起主動關閉，或是 client 發起主動關閉**，但是「通常」都是 client 發起，因此下圖使用 TCP A 與 TCP B 表示，：

![](TCPIP-2.png)

1. TCP A：準備關閉連線，發出 FIN，進入 FIN_WAIT_1 狀態
2. TCP B：收到 FIN，並回傳 ACK，進入 CLOSE_WAIT 狀態，並通知 Application 連線準備關閉
3. TCP A：收到 ACK，進入 FIN_WAIT_2 狀態，並等待對方發出 FIN
4. TCP B：確認 Application 處理完斷線請求，發出 FIN，並進入狀態 LAST_ACK
5. TCP A：收到 FIN，並回傳 ACK，進入 TIME_WAIT 狀態，等待 2MSL 時間後正式關閉連線
6. TCP B：收到 ACK，便直接關閉連線，進入 CLOSED 狀態

名詞解釋：
- ESTABLISHED：表示已完成雙方連線的建立，可開始傳輸資料
- CLOSE_WAIT：等待連線關閉狀態，等待 Application 回應
- LAST_ACK：等待連線關閉狀態，等待對方回傳 ACK 後關閉連線
- FIN_WAIT_1：等待連線關閉狀態，等待對方回傳 ACK
- FIN_WAIT_2：等待連線關閉狀態，等待對方回傳 FIN
- TIME_WAIT：等待2MSL，保證遠端有收到其 ACK 關閉連線 (網路延遲問題)
- CLOSED：已關閉連線
- FIN：FINISH，表示關閉連線的請求
- ACK：Acknowledgement，表示發送的數據已收到無誤

「TCP關閉連線」狀態流程圖

![](TCPIP-3.png)

最後發送 ACK 時，會進入 TIME_WAIT 狀態，要等 2MSL 時間後，這條連接才真正消失，所以從這邊知道「主動關閉連線」的一方會進入 TIME_WAIT 狀態

> 什麼是 MSL 時間
> 最大分段壽命 MSL(Maximum Segment Lifetime)，是 TCP 協定規定封包在網絡中最長生存時間，超出時間後封包就會被丟棄。
>
> RFC793 定義 MSL 為 2 分鐘，不過實際上不同的作業系統可能有不同的設定，以 Linux 為例，通常是 30 秒，2MSL 就是 60 秒

## 為何 TIME_WAIT 狀態要等待 2MSL 的時間呢?

Client/Server 都完成了四次交握，代表 Client/Server 都同意關閉連線，照理說應該可以直接回到 CLOSED 狀態(就像是建立連線時 SYN_SEND 狀態到 ESTABLISH 狀態那樣)，但是這邊我們必須要假設網路傳輸是不可靠的，因為我們無法保證最後傳送的 ACK 對方一定會收到，傳輸過程中可能因為種種原因(例如：網路延遲、丟包等...) 導致對方一直處於 LAST_ACK 狀態下的 SOCKET 因為逾時而未收到 ACK 而重發 FIN，所以 TIME_WAIT 狀態的作用就是用來重發可能遺失的 ACK。

TIME_WAIT 狀態，是為了避免因為網路傳輸的種種原因而造成的 TCP 傳輸不可靠，而 TIME_WAIT 狀態可以最大限度的提升網路傳輸的可靠性。

## 完整 TCP「建立連線」與「關閉連線」之狀態圖

![](TCPIP-4.png)

## 參考資料

- https://dev.twsiyuan.com/2017/09/tcp-states.html
- https://www.gushiciku.cn/pl/p0aJ/zh-tw
