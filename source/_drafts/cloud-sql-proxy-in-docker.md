---
title: 在 Docker Engine 使用 Cloud SQL Auth Proxy
tags:
  - GCP
  - Cloud SQL
  - Google Cloud Platform
categories:
  - GCP
abbrlink: 8ebcb883
---

## 前言

因為之前的 [透過 Auth Proxy 存取 Cloud SQL](https://blog.tonyjhang.tk/posts/8ebcb883/) 的篇幅有點長，所以決定把 Docker Engine 使用 Cloud SQL Auth Proxy 的方法另外寫一篇，在看這篇之前，如果還沒有閱讀過上一篇，可以點擊上面連結，先去看看，再回過頭來接續

<!-- more -->

## Cloud SQL Auth Proxy in a Docker container

建立 Google Service Account
```bash
gcloud iam service-accounts create gsa-cloud-sql \
    --project=demo-123
```

並且賦予權限，以下三個 IAM 權限任一皆可

- roles/cloudsql.client
- roles/cloudsql.editor
- roles/cloudsql.admin

```bash
gcloud projects add-iam-policy-binding demo-123 \
    --member "serviceAccount:gsa-cloud-sql@demo-123.iam.gserviceaccount.com" \
    --role "roles/cloudsql.client" \
    --project=demo-123
```

記得把 Google Service Account 的 JWT 下載到本機

將 JWT 與連線資訊掛載到 container 內，讓容器拿著這把 Key 到 GCP 上做授權驗證
```bash
# Format
docker run -d \
  -v [LOCAL_PATH_TO_KEY_FILE]:/path/to/service-account-key.json \
  -p 127.0.0.1:3306:3306 \
  gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.2 \
  --credentials-file /path/to/service-account-key.json \
  [INSTANCE_CONNECTION_NAME]

# Eample
sudo docker run -d \
  -v /home/tony/key.json:/path/to/service-account-key.json \
  -p 127.0.0.1:3306:3306 \
  gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.2 \
  --credentials-file /path/to/service-account-key.json \
  demo-123:asia-east1:test
```

關於 `-p 127.0.0.1:3306:3306`，如果是要讓 Docker 環境內其他 container 可以存取，需要將 `127.0.0.1` 調整為 `0.0.0.0`，或者直接省略，如：`-p 3306:3306`，因為預設就是 `0.0.0.0`

> Always specify 127.0.0.1 prefix in -p so that the Cloud SQL Auth proxy is not exposed outside the local host. The "0.0.0.0" in the instances parameter is required to make the port accessible from outside of the Docker container.

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

## 參考資料

- https://cloud.google.com/sql/docs/mysql/connect-auth-proxy