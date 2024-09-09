---
title: htpasswd的使用
categories:
  - Linux
abbrlink: c66ce192
date: 2024-01-08 15:21:44
---


## 前言

之前在測試 Traefik BasicAuth 功能時，需要使用 MD5/SHA1/Bcrypt 三種其中一種方式來加密密碼，並寫到設定檔裡

常見的 Nginx/Apache 也有內建網頁的基本身份驗證，這時候也會需要使用 htpasswd 來將使用者的密碼加密並寫進設定檔

下面會介紹如何使用 htpasswd 指令來產生加密密碼

<!--more-->

## htpasswd的使用

其實使用的方式很簡單，只要將使用者的「帳號」與「密碼」丟給 htpasswd，他就會自動幫你加密指定密碼並且整理好格式輸出給你使用，詳細可以參考下面的實際範例

使用 Bcrypt 加密密碼
```
# Format
htpasswd -nb -B [username] [password]

# Example
htpasswd -nb -B tony 1234

# Output
tony:$2y$05$c1xVKSuI8mbPRNll.t.P0OjXC2RShcGww2PBslit0mk.wnatG1cIy
```

使用 MD5 加密密碼
```
# Format
htpasswd -nb -m [username] [password]

# Example
htpasswd -nb -m tony 1234

# Output
tony:$apr1$eRtkDKOy$shi95jDEBupv6F5XcpXyL/
```

使用 SHA1 加密密碼
```
# Format
htpasswd -nb -s [username] [password]

# Example
htpasswd -nb -s tony 1234

# Output
tony:{SHA}cRDtpNCeBiql5KOQsKVyrA0sAiA=
```

## 文後討論

最詳細的使用說明，可以使用 `--help` 來檢視

```
$ htpasswd --help

Usage:
	htpasswd [-cimBdpsDv] [-C cost] passwordfile username
	htpasswd -b[cmBdpsDv] [-C cost] passwordfile username password

	htpasswd -n[imBdps] [-C cost] username
	htpasswd -nb[mBdps] [-C cost] username password
 -c  Create a new file.
 -n  Don't update file; display results on stdout.
 -b  Use the password from the command line rather than prompting for it.
 -i  Read password from stdin without verification (for script usage).
 -m  Force MD5 encryption of the password (default).
 -B  Force bcrypt encryption of the password (very secure).
 -C  Set the computing time used for the bcrypt algorithm
     (higher is more secure but slower, default: 5, valid: 4 to 31).
 -d  Force CRYPT encryption of the password (8 chars max, insecure).
 -s  Force SHA encryption of the password (insecure).
 -p  Do not encrypt the password (plaintext, insecure).
 -D  Delete the specified user.
 -v  Verify password for the specified user.
On other systems than Windows and NetWare the '-p' flag will probably not work.
The SHA algorithm does not use a salt and is less secure than the MD5 algorithm.
```
