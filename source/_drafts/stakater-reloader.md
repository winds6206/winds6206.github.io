---
title: Kubernetes 更新 ConfigMap 或 Secret 自動重啟 Pod
tags:
  - GKE
  - GCP
  - K8s
  - kubernetes
  - stakater
  - reloader
  - opensource
categories:
  - GKE
abbrlink: 1a2eedde
---

## 前言

有在使用 K8s 的朋友，後期應該都會遇到一個問題，內部服務有一些設定檔是使用 ConfigMap 單獨掛出來，或是一些機敏資料是使用 Secret 掛出來，當這些單獨掛出來的資料有異動重新 `kubectl apply`，主要服務卻沒有辦法自動重啟(因為大部分都需要重啟才能吃到新的設定Ex: Nginx)，這時候只能手動將服務砍掉讓它重啟，這使得使用上有那麼一點不便利，如果當要修改的設定檔一多，所耗費的時間也就越多，如果能夠讓設定檔有異動時，服務可以自動重啟，那就再好不過了。

<!-- more -->

為了解決這個問題，下面將要介紹一款開源的套件 Reloader，將它安裝在 K8s 內，並在 `annatation` 多加一些設定，Reloader 就會自動幫你偵測服務所掛載的 Configmap/Secret 資料是否有異動，當有異動時就會幫忙將做 Rolling Update

## Reloader的安裝

安裝 Reloader

```
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
```

在預設下，Reloader 會部署在 `default` 的 namespace 且會幫你監視「所有 namespace」的 `Secret`、`ConfigMap` 是否有異動

> By default, Reloader gets deployed in `default` namespace and watches changes `secrets` and `configmaps` in all namespaces.

安裝後，在 K8s `default` namespace 會多一個 stakater-reloader 服務

## Reloader的使用

安裝後，我們必須告訴 Reloader，要請他幫我們監視哪個服務所掛載的 `Secret` 或 `ConfigMap`，

假設 Deployment 名字是 foo，而他掛一個 ConfigMap 名稱叫 foo-configmap，然後在 deployment Object 的 annotation 加上主要設定，如下

```yaml
kind: Deployment
metadata:
  annotations:
    configmap.reloader.stakater.com/reload: "foo-configmap"
```

同樣的，如果要監視 `Secret` 也是一樣的方式，只是 annotation 要稍微改一下，由 `configmap.reloader.stakater.com/reload` 改成 `secret.reloader.stakater.com/reload`，冒號後面的值則是帶入 `Secret` 名稱

```yaml
kind: Deployment
metadata:
  annotations:
    secret.reloader.stakater.com/reload: "foo-secret"
```

如果 Deployment 同時掛載多個 `Secret` 或 `ConfigMap`，則以逗號分開，例如

```yaml
kind: Deployment
metadata:
  annotations:
    configmap.reloader.stakater.com/reload: "foo-configmap,bar-configmap,baz-configmap"
```

當然也可以讓 Reloader 自行判斷，`reloader.stakater.com/auto: "true"` 會自行判斷 Deployment 是否有掛載 ConfigMap 或 Secret 當掛載的資料有異動時會自動重啟 Pod

> `reloader.stakater.com/auto: "true"` will only reload the pod, if the configmap or secret is used (as a volume mount or as an env) in `DeploymentConfigs/Deployment/Daemonsets/Statefulsets`

```yaml
kind: Deployment
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

## Reloader的實際應用

如果懶得自己寫 YAML 的朋友，可以拿下面的 Nginx 範例部署到環境內做測試，Deployment/ConfigMap 都部署後，可以試著修改 ConfigMap 中 Nginx 設定檔內容，在重新部署，就會發現 Nginx 神奇的自己重新啟動。

Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-reloader-test
  namespace: default
  labels:
    app: nginx-reloader-test
    annotations:
      configmap.reloader.stakater.com/reload: "nginx-config"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-reloader-test
  template:
    metadata:
      labels:
        app: nginx-reloader-test
    spec:
      restartPolicy: Always
      containers:
      - name: nginx-reloader-test
        image: nginx:1.21.1
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-conf-volume
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-conf-volume
        configMap:
          name: nginx-config
```

ConfiMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen       80;
        server_name  localhost;
        root   /usr/share/nginx/html;
        index  index.php index.html index.htm;

        # redirect server error pages to the static page /50x.html
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        location / {
            try_files $uri $uri/ =404;
        }
    }
```

## 文後討論

官方 GitHub 底下還有蠻多可以調整的參數，如果有特殊需求的朋友，可以到 [官方GitHub](https://github.com/stakater/Reloader) 去挖寶，或許會找到你所需要，尤其是在 NOTES 的區塊，可以考慮好好讀一下

https://github.com/stakater/Reloader#notes

## 參考資料

- https://github.com/stakater/Reloader
