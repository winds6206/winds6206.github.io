---
title: Kubernetes Network Policy
tags:
  - GCP
  - GKE
  - Google Cloud Platform
  - K8s
  - Kubernetes
categories:
  - GKE
date: 2024-01-16 13:51:16
---


## 前言

在 K8s 中，預設所有 Pod 不管是在哪個 Namespace，預設都是相通，若是想要進一步限制 Pod 之間的存取，可以使用 Network Policy 來達成。

在 Google Kubernetes Engine 的環境中，可以自行啟用「Calico Kubernetes Network 政策」，即可輕鬆簡單的直接使用。

<!--more-->

## Network Policy 描述檔

基本範例

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx-vts
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          repo: sugar
    - namespaceSelector:
        matchLabels:
          department: rd
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    ports:
    - protocol: TCP
      port: 9453
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 9527
```

### 分段解析

#### Part 1

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx-vts
  policyTypes:
  - Ingress
  - Egress
...
```

在 `default` Namespace 中只要帶有 Label 為 `app=nginx-vts` 的 pod，接會被套上 Network Policy，並且進(ingress) 出(egress)流量皆會被限制。

> 如果沒有設定 `spec.podSelector`，代表會套用到 `default` Namespace 中的所有 pod

至於進(ingress) 出(egress)流量的限制細節，需要再往後設定

#### Part 2

Ingress 流量的詳細設定

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx-vts
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          repo: sugar
    - namespaceSelector:
        matchLabels:
          department: rd
    # IP 範圍，而非 IP 封鎖
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    ports:
    - protocol: TCP
      port: 9453
...
```

Ingress from：管理從外部進來的流量，也就「允許」進來 Pod 的流量

允許存取的來源白名單：

- 允許 `default` Namespace 中帶有 `repo=sugar` Label 的 Pod
- 允許帶有 `department=rd` Label 的 Namespace 中所有 Pod
- 來源 IP 範圍 172.17.0.0/16 (不含 172.17.1.0/24)

綜合 Part 1 的解析，會變成如下：

- 允許 `default` Namespace 中帶有 `repo=sugar` Label 的 Pod，存取 `default` Namespace 中帶有 `app=nginx-vts` 標籤 Pod 的 9453 Port
- 允許帶有 `department=rd` Label 的 Namespace 中所有 Pod，存取 `default` Namespace 中帶有 `app=nginx-vts` 標籤 Pod 的 9453 Port
- 允許來源 IP 為 172.17.0.0/16(但排除 172.17.1.0/24)，存取 `default` Namespace 中帶有 `app=nginx-vts` 標籤 Pod 的 9453 Port

#### Part 3

Egress 流量的詳細設定

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx-vts
  policyTypes:
  - Ingress
  - Egress
...
  egress:
  - to:
    # IP 範圍，而非 IP 封鎖
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 9527
```

Egress to：管理從內部出去的流量，也就是「允許」從 Pod 的出去到達的目的流量

「允許」從 Pod 的出去到達的目的流量白名單：

- 只允許對外連線(egress) 到 IP 範圍 10.0.0.0/24 的 通訊協定 TCP 且目的 Port 為 9527。

綜合 Part 1 的解析，會變成如下：

「在 `default` 的命名空間中只要帶有 Label 為 `app=nginx-vts` 的 pod，只允許對外連線(egress) 到 IP 範圍 10.0.0.0/24 的 通訊協定 TCP 且目的 Port 為 9527」

#### 結論

從上述的解析中可以推論，不管是 Ingress 或是 Egress，對於設定上皆有三種白名單的細部控制：`podSelector`、`namespaceSelector`、`ipBlock`

## 預設政策(default policy)

基本概念(重要)：

- 若要選擇所有的 pod，則將 `podSelector` 設定為 `{}`
- 若是設定 `policyType`，就表示要全部拒絕該種連線(ingress/egress) 的流量
- 若要打開上面的限制(也就是白名單)，則要在 `spec.ingress` 或 `spec.egress` 中設定

綜合上述，可以推理出四種情況：

- 拒絕從外面進來的流量，要在 `spec.policyTypes` 中包含 Ingress，並且略過 ingress 細部設定

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

- 拒絕從內部出去的流量，要在 `spec.policyTypes` 中包含 Egress，並且略過 egress 細部設定

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

- 接受從外面進來的網路流量，要在 `spec.policyTypes` 中包含 Ingress，並將 `spec.ingress` 設定為 `{}`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}
```

- 接受從內部出去的網路流向，要在 `spec.policyTypes` 中包含 Egress，並將 `spec.egress` 設定為 `{}`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - {}
```

在理解完上述情況後後，記得帶上 Namespace 的概念進去，這樣才知道該條政策的作用範圍：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
  namespace: default
spec:
  podSelector: {}
...
```

在 `default` Namespace 中所有的 pod(`podSelector: {}`)，皆會被套上 Network Policy，並且進(ingress) 出(egress)流量皆會被限制。

## 更多的使用場景

下述參考資料中，有更多的使用場景可以參照：

https://kubernetes.feisky.xyz/concepts/objects/network-policy#shi-yong-chang-jing

### 只允許同一個 Namespace 間的 Pod 相互訪問

只允許 `monitoring` Namespace 間的 Pod 可以相互訪問，其他 Namespace 的 Pod 無法訪問 `monitoring` Namespace 的 Pod

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: monitoring
  name: deny-other-namespaces
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
```

### 禁止同一個 Namespace 間的 Pod 相互訪問，但是跨 Namespace 不在此限

禁止 `default` Namespace 間的 Pod 相互訪問，但 `default` Namespace 可以訪問其他 Namespace 的 Pod，其他 Namespace 的 Pod 也可以訪問 `default` Namespace 的 Pod 

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

## 參考資料

- https://godleon.github.io/blog/Kubernetes/k8s-Network-Policy-Overview/
- https://kubernetes.feisky.xyz/concepts/objects/network-policy#shi-yong-chang-jing
