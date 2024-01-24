---
title: 透過 Auth Proxy 存取 Cloud SQL
tags:
  - GCP
  - Cloud SQL
  - Google Cloud Platform
categories:
  - GCP
abbrlink: 3850bdf7
---

## 前言

一般來說，要連線到 Could SQL 需要透過 Proxy 與 GCP 做授權驗證，而該代理工具稱為「Cloud SQL Auth Proxy」

Cloud SQL Auth Proxy 是存取 Could SQL 的連線工具，讓我們無須額外設定網路授權或 SSL 的相關設定，提供較安全的存取方式。

<!-- more -->

使用 Cloud SQL Auth Proxy 的好處大致有三點：
- 安全的加密連線
- 透過 IAM 的簡易連線授權
- 自動更新 OAuth 2.0 Token

有興趣深入了解，可以參考[官方文件](https://cloud.google.com/sql/docs/mysql/sql-proxy)

## 大致設定流程

大前提：
- Google Service Account 簡稱 GSA
- Kubernetes Service Account 簡稱 KSA
- Project ID 為 demo-123
- Cloud SQL 使用的是 MySQL

使用 Google Service Account/E-mail 加上 Cloud SQL Auth Proxy 來做授權驗證，簡單來說，就是透過 Google Service Account/E-mail 的 IAM 權限管控，來決定使用者是否能夠存取到 Cloud SQL 的服務，接著再使用 Cloud SQL 裡面的使用者帳號與 SQL 做連線操作。

主要設定步驟如下

1. 創建 Google Service Account 並賦予 IAM 權限 → roles/cloudsql.client

以下三個 IAM 權限任一皆可

- roles/cloudsql.client
- roles/cloudsql.editor
- roles/cloudsql.admin

2. 下載 Google Service Account 的 JWT
3. 將 JWT 交付給 Cloud SQL Auth Proxy 做授權驗證
4. 使用 Cloud SQL 的使用者帳號與 Cloud SQL Auth Proxy 做連線

## 本機使用 Cloud SQL Auth Proxy
 
這種方式是透過 Auth Proxy 與 Cloud SQL 建立起一個 Tunnel，本機就可以直接與 Auth Proxy 連線存取到 Cloud SQL

有兩種方式可以達到目的

- 使用 Google Service Account
- 使用 E-mail

### 方法一：使用 Service Account

下載相對應作業系統的 Cloud SQL Auth Proxy，並按照官方的方式修改權限

https://cloud.google.com/sql/docs/mysql/connect-admin-proxy#cloud-sql-auth-proxy-docker-image_1

建立 Google Service Account
```bash
gcloud iam service-accounts create gsa-cloud-sql \
    --project=demo-123
```

並且賦予權限
> IAM 權限請參考前一節
```bash
gcloud projects add-iam-policy-binding demo-123 \
    --member "serviceAccount:gsa-cloud-sql@demo-123.iam.gserviceaccount.com" \
    --role "roles/cloudsql.client" \
    --project=demo-123
```

記得把 Google Service Account 的 JWT 下載到本機

使用 TCP Socket 的方式啟用 Cloud SQL Auth Proxy
```bash
# Format
./cloud-sql-proxy --address 0.0.0.0 --port 3306 \
    [INSTANCE_CONNECTION_NAME] \
    --credentials-file [PATH_TO_KEY_FILE]

# Example
./cloud-sql-proxy --address 0.0.0.0 --port 3306 \
    demo-123:asia-east1:test \
    --credentials-file ./demo-123-d617764af0fb.json
```

連線名稱(connectionName) 的查詢方式，或者直接到網頁看
```bash
# Format
gcloud sql instances describe [Instance_Name] --project=[Project_ID]

# Example
gcloud sql instances describe test --project=demo-123
```

最後使用 Cloud SQL 的使用者帳號與 Cloud SQL Auth Proxy 做連線驗證，也可以使用第三方工具來連線測試，例如：MySQL WorkBench, phpMyAdmin...

```bash
mysql -u [USERNAME] -p --host 127.0.0.1
```

### 方法二：使用 E-mail

改成使用自己的 E-mail 帳號賦予的 IAM 權限來做授權驗證，確認此帳號是否有存取 cloudSQL 的權限，所以**在 proxy 在與 GCP 做授權驗證前，我們必須手動使用 Cloud SDK 來登入 Google Cloud 並透過瀏覽器來確認 Cloud Identity credentials**，但是一般來說本機都已經有做過此動過了

> uses the Cloud SDK to log in to Google Cloud and enters his Cloud Identity credentials through the web browser.

如果還未執行 Cloud Identity credentials，可以使用下面指令，這部份就不多做說明，當作大家都已經知道如何登入
```bash
gcloud auth login
```

將自己的 E-mail 帳號新增 IAM 權限

```bash
gcloud projects add-iam-policy-binding demo-123 \
    --member "user:tony@gmail.com" \
    --role "roles/cloudsql.client" \
    --project=demo-123
```

產生應用程式使用的 credential，下了該指令後，會有網頁流程要跑，其實就是把你的 E-mail 的權限做成 json 檔，然後 Proxy 程式預設會去拿這支檔案來用
```bash
gcloud auth application-default login
```

這將會在 `${HOME}/.config/gcloud` 產生一個 json 檔 「application_default_credentials.json」，如要撤銷可以使用 `gcloud auth application-default revoke`，此時檔案將會被移除並撤銷

接著直接在本機使用下面指令，可以發現這次的指令是不需要 `--credentials-file`，因為我們需要讓系統自動到 `${HOME}/.config/gcloud` 取得授權驗證

```bash
# Format
./cloud-sql-proxy --port 3306 [INSTANCE_CONNECTION_NAME]

# Example
./cloud-sql-proxy --port 3306 demo-123:asia-east1:test
```

## Cloud SQL Auth Proxy in K8s

在 GKE 內的 Pod 若是要存取 Cloud SQL，一樣是透過 Cloud SQL Auth Proxy 來做代理的動作，官方建議將 Auth Proxy 以 sidecar 的形式運行在需要使用的 Pod 上，並且依照不同服務給予不同的  Google Service Account，當然如果有特殊理由，也是可以多個 Pod 使用同一個 Cloud SQL Auth Proxy 進行存取。

在 GKE 中使用 Google Service Account，原廠已經建議使用 Workload-Identity 的方式進行綁定，不建議再將 JWT 下載後以 Secret 方式掛載，對於還不知道 GKE Workload-Identity 或者想更進一步了解使用方式的，可以參考我的[另外一篇](https://blog.tonyjhang.tk/posts/7d4918c/#more)，裡面有更詳細的介紹。

Google Service Account 的設定，前面已經有步驟，這邊不再重複贅述，直接從 Workload-Identity 中 KSA 和 GSA 綁定步驟開始

建立 K8s ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ksa-cloud-sql-proxy
  namespace: default
  annotations:
    iam.gke.io/gcp-service-account: gsa-cloud-sql-proxy@demo-123.iam.gserviceaccount.com
```

將 KSA(ksa-cloud-sql-proxy) 與 GSA(gsa-cloud-sql-proxy) 進行綁定
```bash
gcloud iam service-accounts add-iam-policy-binding gsa-cloud-sql-proxy@demo-123.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:demo-123.svc.id.goog[default/ksa-cloud-sql-proxy]" \
    --project=demo-123
```

佈署 Cloud SQL Auth Proxy，這邊沒有以 sidecar 方式呈現，如果要以 sidecar 運行的話，可以再自己加以修改
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloud-sql-proxy
  namespace: default
  labels:
    app: cloud-sql-proxy
    product: demo
spec:
  type: ClusterIP
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: cloud-sql-proxy
    product: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloud-sql-proxy
  namespace: default
  labels:
    app: cloud-sql-proxy
    product: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloud-sql-proxy
      product: demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: cloud-sql-proxy
        product: demo
    spec:
      restartPolicy: Always
      serviceAccountName: ksa-cloud-sql-proxy
      containers:
      - name: cloud-sql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:1.33.14
        imagePullPolicy: Always
        ports:
          - containerPort: 3306
        command:
          - /cloud_sql_proxy
          - -instances=demo-123:asia-east1:test=tcp:0.0.0.0:3306
        env:
          - name: TZ
            value: Asia/Taipei
      nodeSelector:
        iam.gke.io/gke-metadata-server-enabled: "true"
```

最後用 phpMyAdmin 測試連線情況
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: phpmyadmin
    repo: demo
  name: phpmyadmin
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
      repo: demo
  template:
    metadata:
      labels:
        app: phpmyadmin
        repo: demo
    spec:
      containers:
      - env:
        - name: PMA_HOST
          value: cloud-sql-proxy.default.svc.cluster.local # 填入 cloud-sql-proxy 的 service 
        - name: PMA_PORT
          value: "3306"
        image: phpmyadmin:5.1.1
        imagePullPolicy: Always
        name: phpmyadmin
      restartPolicy: Always
```

## 文後討論

如果對於 GKE 上使用 Workload-Identity 不懂的，不妨先去[補一下](https://blog.tonyjhang.tk/posts/7d4918c/#more)一下 

官方文件在掛載 KSA 的 deployment 描述檔有加上 `spec.nodeSelector`，來確保服務正常跑在有啟用 Workload-Identity 的 Node 上

```yaml
spec:
  nodeSelector:
    iam.gke.io/gke-metadata-server-enabled: "true"
```

## 參考資料

- https://cloud.google.com/sql/docs/mysql/connect-auth-proxy
- https://cloud.google.com/sql/docs/mysql/sql-proxy#benefits_of_the
- https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity