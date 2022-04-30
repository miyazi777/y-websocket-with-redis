# 概要
y-websocketがそのままの状態ではy-redisと連携していなかった為、levelDBとの接続部分をy-redisに変更したバージョンです。
元のリポジトリ
https://github.com/yjs/y-websocket

# ローカルでの動作確認
## git clone
```
git clone https://github.com/miyazi777/y-websocket-with-redis
cd y-websocket-with-redis
```

## 起動
y-websocketサーバとredisのコンテナが起動します。
```
docker-compose up -d
```

停止する時はstopコマンドで。
```
docker-compose stop
```

# clour runへのデプロイ 
## git clone
```
git clone https://github.com/miyazi777/y-websocket-with-redis
cd y-websocket-with-redis
```

## 環境変数設定
下記を環境に合わせて設定して下さい。
```
export GCP_PROJECT=<gcpのproject idを設定して下さい>
export GCP_REGION=<asia-northeast1など、regionを設定して下さい>
export NETWORK=<vcp名を設定して下さい>
```

## redis作成
```
gcloud redis instances create cloudrun-redis-test \
  --size=1 \
  --network projects/$GCP_PROJECT/global/networks/$NETWORK \
  --redis-config maxmemory-policy=volatile-lru \
  --region $GCP_REGION
```

## サーバーレスvcpアクセス作成
```
gcloud compute networks vpc-access connectors create redis-vcp-connector \
  --network $NETWORK \
  --region $GCP_REGION \
  --range "10.8.0.0/28"
```

## cloud runへデプロイ
### redisの接続情報を取得
```
export REDISHOST=$(gcloud redis instances describe cloudrun-redis-test --region $GCP_REGION --format "value(host)")
export REDISPORT=$(gcloud redis instances describe cloudrun-redis-test --region $GCP_REGION --format "value(port)")
```

### サーバのデプロイ
```
gcloud run deploy y-socket-test \
  --source=. \
  --timeout=60m \
  --vpc-connector redis-vcp-connector \
  --platform managed \
  --region $GCP_REGION \
  --allow-unauthenticated \
  --set-env-vars HOST=0.0.0.0 \
  --set-env-vars REDIS_HOST=$REDISHOST \
  --set-env-vars REDIS_PORT=$REDISPORT \
  --set-env-vars REDIS_DB=1
```

デプロイが完了すると最後にアプリへ接続する為のURLが出てくるのでこれをクライアント側の接続先に指定して下さい。<br>
https://部分だけwss://に変更して下さい。

#### 例
https://y-socket-test-aoytzwylua-an.a.run.app -> wss://y-socket-test-aoytzwylua-an.a.run.app

クライアント側のコードで接続先を指定している以下のような部分を変更して下さい。
```
const wsProvider = new WebsocketProvider('ws://localhost:1234', 'my-roomname', doc)
```

## 後片付け
### cloud run サービス削除
```
gcloud run services delete y-socket-test --region $GCP_REGION
```

### サーバーレスVCPコネクタ削除
```
gcloud compute networks vpc-access connectors delete redis-vcp-connector --region $GCP_REGION
```

### redis削除
```
gcloud redis instances delete cloudrun-redis-test --region $GCP_REGION
```

### 確認
```
gcloud run services list --region $GCP_REGION
gcloud compute networks vpc-access connectors list --region $GCP_REGION
gcloud redis instances list --region $GCP_REGION
```


