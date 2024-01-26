---
title: 在 GKE 內使用 Cert-Manager
tags:
  - GCP
  - GKE
  - Google Cloud Platform
  - K8s
  - Kubernetes
categories:
  - GKE
abbrlink: c0478787
---

## 前言

cert-manager 可以幫助我們在使用 kubernetes 時，可以自動簽發 Let's Encrypt 憑證，並且搭配 Ingress 做使用。

> Let's Encrypt 憑證有效期限為三個月，三個月到期會自動在重新簽發展延

<!-- more -->

本文章使用環境為 GKE 搭配 GKE Ingress 並且使用 HTTP-01 Challenge

## 安裝 Cert-Manager

首先需要在 Cluster 環境內安裝 Cert-Manager 核心元件

安裝 Cert-Manager，如果有新版本，路徑會不同，這部份可以直接上官方文件查詢
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
```

檢查安裝元件
```bash
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-75997f4b44-jsq7z              1/1     Running   0          4m38s
cert-manager-cainjector-769785cd7b-95fcp   1/1     Running   0          4m38s
cert-manager-webhook-6bc9944d78-dnggk      1/1     Running   0          4m38s
```

## 前製作業

- 先預約 GKE Ingress 用的外網 IP
- 設定DNS A紀錄

確認 cert-manager 要與哪種 Ingress Controller 搭配，可參考以下資料：

https://cert-manager.io/docs/getting-started/

> 如果要使用 wildcard 憑證只能用 DNS-01 challenge

## 佈署建議指南

本章測試原始碼：https://github.com/winds6206/cert-manager

### 佈署 application 內的 Nginx

佈署服務
```bash
cd ./application/nginx
kubectl apply -f .

cd ./application/nginx-v2
kubectl apply -f .
```

### 佈署 Ingress 和 NodePort Service

以 80 為主，先確保可以正確導向到服務

修改 Ingress 描述檔，並且佈署，域名記得跟著自身測試環境修改：

- `kubernetes.io/ingress.global-static-ip-name` 改成自己預約的IP名稱
- `kubernetes.io/ingress.allow-http` 改成 true
- `networking.gke.io/v1beta1.FrontendConfig` 新增註解
- `cert-manager.io/issuer` 新增註解
- 下述片段新增註解
```yaml
#tls:
#  - secretName: web-ssl
#    hosts:
#      - cert-manager-demo.example.com
#      - cert-manager-demo-v2.example.com
```

整體如下：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cert-manager-demo-external-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "cert-manager-demo-external-ingress"
    kubernetes.io/ingress.allow-http: "true" # default is true
#    networking.gke.io/v1beta1.FrontendConfig: "cert-manager-demo-external-ingress-http2https"
    ingressClassName: "gce" # This tells Google Cloud to create an External Load Balancer to realize this Ingress
#    cert-manager.io/issuer: letsencrypt-production
spec:
#  tls:
#    - secretName: web-ssl
#      hosts:
#        - cert-manager-demo.example.com
#        - cert-manager-demo-v2.example.com
  rules:
    - host: cert-manager-demo.example.com
      http:
        paths:
        - path: /*
          pathType: ImplementationSpecific
          backend:
            service:
              name: cert-manager-demo-for-external-ingress
              port:
                number: 80
    - host: cert-manager-demo-v2.example.com
      http:
        paths:
        - path: /*
          pathType: ImplementationSpecific
          backend:
            service:
              name: cert-manager-demo-v2-for-external-ingress
              port:
                number: 80
```

佈署服務
```bash
cd ./ingress
kubectl apply -f svc-external-ingress.yaml
kubectl apply -f cert-manager-demo-external-ingress.yaml
```

### 佈署 issuer 和 secret

80 可以正常運行後，再將 issuer 佈署下去做憑證簽發，確認 80 和 443 是否可以分開正常運行

先修改 Ingress 描述檔，將 cert-manager 相關的設定註解移除，並且重新佈署

- `cert-manager.io/issuer` 註解移除
- 下述片段註解移除，域名記得更換
```yaml
tls:
  - secretName: web-ssl
    hosts:
      - cert-manager-demo.example.com
      - cert-manager-demo-v2.example.com
```

整體如下：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cert-manager-demo-external-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "cert-manager-demo-external-ingress"
    kubernetes.io/ingress.allow-http: "true" # default is true
#    networking.gke.io/v1beta1.FrontendConfig: "cert-manager-demo-external-ingress-http2https"
    ingressClassName: "gce" # This tells Google Cloud to create an External Load Balancer to realize this Ingress
    cert-manager.io/issuer: letsencrypt-production
spec:
  tls:
    - secretName: web-ssl
      hosts:
        - cert-manager-demo.example.com
        - cert-manager-demo-v2.example.com
  rules:
    - host: cert-manager-demo.example.com
      http:
        paths:
        - path: /*
          pathType: ImplementationSpecific
          backend:
            service:
              name: cert-manager-demo-for-external-ingress
              port:
                number: 80
    - host: cert-manager-demo-v2.example.com
      http:
        paths:
        - path: /*
          pathType: ImplementationSpecific
          backend:
            service:
              name: cert-manager-demo-v2-for-external-ingress
              port:
                number: 80
```

調整 issuer 描述檔，將 `spec.acme.email` 填入自己正確的 E-mail，再將 issuer 和 secret 佈署下去

```bash
kubectl apply -f issuer.yaml
kubectl apply -f secret.yaml
```

憑證簽發狀態可以使用 `kubectl describe issuers.cert-manager.io [Issuer_Name]` 去確認，也可以使用 `kubectl get certificate` 去確認是否 READY

確認簽證後，看看 80 跟 443 能不能正常存取網頁

### 附加 https 跳轉功能

最後將 http 自動跳轉 https 功能加上

```bash
cd ./ingress
kubectl apply -f frontend-config.yaml
```

修改 Ingress 描述檔，並且重新佈署

- `networking.gke.io/v1beta1.FrontendConfig` 移除註解

整體如下：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cert-manager-demo-external-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "cert-manager-demo-external-ingress"
    kubernetes.io/ingress.allow-http: "true" # default is true
    networking.gke.io/v1beta1.FrontendConfig: "cert-manager-demo-external-ingress-http2https"
    ingressClassName: "gce" # This tells Google Cloud to create an External Load Balancer to realize this Ingress
    cert-manager.io/issuer: letsencrypt-production
spec:
  ...
```

測試 80 是否可以自動跳轉 443，這樣整個功能就完成囉

## Q&A

- Issuer vs ClusterIssuer
- Let's Encrypt Issuer production vs staging
- Issuer 中 E-mail 的作用

### Issuer 和 ClusterIssuer 差異

cert-manager 分為兩種 Issuer，分別為 Issuer 和 ClusterIssuer。

兩者的差異就在於 Issuer 只能在單一 Namespace 使用，而 ClusterIssuer 屬於 cluster-wide 也就是說每個 Namespace 都可以 reference 到。

兩者於 Manifest 的寫法差異僅在於 `kind` 與 `namespace`，如下參考

Issuer
```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-production
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: tony@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-production
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          name: example-ingress
```

ClusterIssuer 
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: tony@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-production
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          name: example-ingress
```

### Let's Encrypt Issuer production 和 staging 差異

Let's Encrypt Issuer 分為 production 和 staging，兩者最大的差別在於 rate limits，production 有相當嚴格的 rate limits，如果是在測試階段很可能不小心就頂到限制，所以在測試階段建議先以 staging 來測試，確定沒問題後，在切換到 production。

使用 staging 簽發出來的憑證依然屬於 untrusted certificates
> Note that you'll see a warning about untrusted certificates from the staging issuer, but that's totally expected.

production 和 staging 的 ACME server URL 如下：

- production：https://acme-v02.api.letsencrypt.org/directory
- staging：https://acme-staging-v02.api.letsencrypt.org/directory

### Issuer 中 E-mail 的作用

憑證註冊時會使用該組 E-mail 做註冊，並且在憑證即將到期時通知該組 E-mail

> This email is required by Let's Encrypt and used to notify you of certificate expiration and updates.

### Issuer, secret 與 Ingress 之間如何產生關聯

在 GKE Ingress 的 Manifest 中有以下兩段：

- `metadata.annotations.cert-manager.io/issuer`
- `spec.tls.secretName`

這兩段分別對應 Issuer 和 secret，當我們將 Issuer 和 secret 佈署後，在 certificate object 中會自動建立一個與 secret 同名的物件，透過 `describe certificate` 就可以發現到，系統會自動將先前建立的物件進行關聯。

```bash
kubectl describe certificate [Certificate_Name]
```

```yaml
省略...

Spec:
  Dns Names:
    ingress-nginx-demo.twister5-tc.website
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       Issuer
    Name:       ingress-nginx-demo-external-ingress-letsencrypt-production
  Secret Name:  ingress-nginx-demo-external-ingress-ssl

省略...
```

從這邊我們可以很清楚的看到，`Spec.Issuer Ref.Name` 與 `Spec.Secret Name` 是有順利關聯起來

當 Issuer 順利簽發出憑證後，會依據自身的 Manifest `spec.acme.privateKeySecretRef.name` 建立相對應名稱的 secret 並且將憑證內容儲存於此。

前面有提到 Ingress 的描述檔內也有 Issuer 和 secret 的資訊，自然而然三者就會互相進行關聯。

## 參考資料

- https://cert-manager.io/docs/installation/kubectl/
- https://cert-manager.io/docs/getting-started/
- https://cert-manager.io/docs/tutorials/getting-started-with-cert-manager-on-google-kubernetes-engine-using-lets-encrypt-for-ingress-ssl/
