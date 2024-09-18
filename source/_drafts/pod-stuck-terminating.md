---
title: Kubernetes Pod 卡在 Terminating 狀態
categories:
  - Kubernetes
abbrlink: 5200c977
---

## 前言

Kubernetes 的 Pod 有時候因為一些不正常的流程或者相依的服務有問題，可能會導致 Pod 卡在 Terminating 的狀態，也無法用一般的方式來刪除，此時會需要使用強制刪除的方式

## 處理方式

強制刪除 Pod，於刪除指令加上`--grace-period=0 --force`

```bash
kubectl delete pod -n [NAMESPACE] [PODNAME] --grace-period=0 --force
```

假設同一個 Namespace 內有多個卡 Terminating 的 Pod，也可以利用 for 迴圈一次刪除

```bash
for p in $(kubectl get pods -n [NAMESPACE] | grep Terminating | awk '{print $1}'); do kubectl delete pod $p --grace-period=0 --force -n [NAMESPACE];done
```

## 參考資料

- https://stackoverflow.com/questions/35453792/pods-stuck-in-terminating-status