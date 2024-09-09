---
title: 'SSH 連線出現「WARNING:REMOTE HOST IDENTIFICATION HAS CHANGED!」'
tags:
  - SSH
categories:
  - Linux
abbrlink: 67cc021f
date: 2021-08-09 10:48:45
---


## 前言

在 Linux 環境中，我們常常使用 SSH 來與目標主機做連線，有時候因為目標主機的異動而導致我們後續連線時會出現 WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED這樣子的警示訊息，導致我們無法順利連線，為何會有這樣子的訊息，又該如何解決，可以參考以下內容

<!--more-->

## 連線原理

SSH 連接遠端主機時，會檢查主機的 Public key，如果是第一次連線該主機，會顯示該主機的 public key fingerprint，詢問使用者信任該主機並繼續連線

> A public key fingerprint is a short sequence of bytes used to identify a longer public key.

```text
The authenticity of host '192.168.26.11 (192.168.26.11)' can't be established.
RSA key fingerprint is a3:ca:ad:95:a1:45:d2:57:3a:e9:e7:75:a8:4c:1f:9f.
Are you sure you want to continue connecting (yes/no)?
```

當選擇 yes，就會將該主機的 Public key 追加到本機 `~/.ssh/known_hosts` 中，當第二次連到該台主機時，就不會再出現相關訊息

## 解決方式

這樣子的 Public key 檢查是一個重要的安全機制，可以防範中間人攻擊等問題，但有時候因爲某些原因，導致該 IP 的 Public key 改變了，這時候使用 SSH 連線時，會出現以下錯誤訊息

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that the RSA host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
e9:0c:36:89:7f:3c:07:71:09:5a:9f:28:8c:44:e9:05.
Please contact your system administrator.
Add correct host key in /home/tonyjhang/.ssh/known_hosts to get rid of this message.
Offending key in /home/tonyjhang/.ssh/known_hosts:81
RSA host key for 192.168.26.11 has changed and you have requested strict checking.
Host key verification failed.
```

有三種解決方式

1. 根據上述訊息，將第 81 行刪除，重新連線，主機會重新傳送一份 public key fingerprint

```text
Offending key in /home/tonyjhang/.ssh/known_hosts:81
```

2. 把整個 `~/.ssh/known_hosts` 刪除，這是最「不建議」的做法，可能會導致自動化失效或連線到其他主機的問題
3. 使用 `ssh-keygen -R` 的指令，將 `~/.ssh/known_hosts` 內的所有資訊，轉存一份到 `~/.ssh/known_hosts.old`，並且將 `~/.ssh/known_hosts` 內的目標主機項目移除，使用方式如下

```bash
$ ssh-keygen -R 192.168.26.11

# Host 192.168.26.11 found: line 60
/Users/tony_jhang/.ssh/known_hosts updated.
Original contents retained as /Users/tony_jhang/.ssh/known_hosts.old
```

> Original contents retained as /Users/tony_jhang/.ssh/known_hosts.old → 原始內容有被保存到 known_hosts.old

## 參考資料

- https://en.wikipedia.org/wiki/Public_key_fingerprint
