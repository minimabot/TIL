# KCL シーケンス集

## 1. 起動と通常処理

```mermaid
sequenceDiagram
    participant App as Spring アプリ
    participant S as Scheduler
    participant D as DynamoDB<br/>Lease テーブル
    participant C as ShardConsumer
    participant R as RecordProcessor
    participant K as Kinesis
    participant B as 業務用 DynamoDB

    App->>S: scheduler.run()
    S->>D: lease を取得・確認
    D-->>S: 担当 shard と checkpoint
    S->>C: shard ごとに consumer を作成
    C->>R: initialize(checkpoint)
    R-->>C: 初期化完了

    loop lease を保持している間
        par lease の維持
            S->>D: lease renew
        and レコード処理
            C->>K: GetRecords
            K-->>C: record batch
            C->>R: processRecords(batch)
            R->>B: 業務データ更新
            B-->>R: 成功
            R->>D: checkpoint
        end
    end
```

## 2. Downstream DynamoDB の一時障害

```mermaid
sequenceDiagram
    participant C as ShardConsumer
    participant R as RecordProcessor
    participant B as 業務用 DynamoDB
    participant D as DynamoDB<br/>Lease テーブル

    C->>R: processRecords(batch)
    R->>B: 業務データ更新
    B-->>R: timeout / throttling / error

    loop 再試行
        R->>B: exponential backoff 後に再試行
        alt 更新成功
            B-->>R: 成功
        else 継続して失敗
            B-->>R: 失敗
        end
    end

    alt 再試行成功
        R->>D: checkpoint
    else 再試行上限到達
        Note over R,D: 失敗した record より先へ checkpoint しない
        Note over R,D: worker 停止、再起動、または DLQ/inbox へ保存
    end
```

## 3. Graceful lease handoff: 再配置または Pod 終了

```mermaid
sequenceDiagram
    participant Old as 既存 Worker
    participant S as Scheduler
    participant R as RecordProcessor<br/>(対象 shard)
    participant D as DynamoDB<br/>Lease テーブル
    participant New as 新しい Worker

    Note over Old,New: 再配置では対象 shard のみ<br/>Pod 終了では全保持 shard に対して実行

    Old->>S: graceful handoff リクエストを検知
    S->>R: shutdownRequested()
    R->>D: 最後の安全な位置まで checkpoint
    D-->>R: checkpoint 完了
    S->>D: lease transfer / release
    D-->>New: 新しい owner に lease を割り当て
    New->>D: checkpoint を取得
    D-->>New: 最後の checkpoint
    Note over New: checkpoint の次の sequence から処理
```

## 4. Pod 異常終了または lease 喪失

```mermaid
sequenceDiagram
    participant Old as 既存 Worker
    participant D as DynamoDB<br/>Lease テーブル
    participant New as 新しい Worker
    participant R as 新しい RecordProcessor

    Old--x Old: OOM / SIGKILL / ネットワーク切断<br/>または lease renew 失敗
    Note over D: lease renew 停止
    Note over D: failover time 経過

    New->>D: 期限切れ lease を取得
    D-->>New: lease と最後の成功 checkpoint
    New->>R: initialize(checkpoint)
    Note over R: checkpoint の次から再処理<br/>重複処理の可能性あり
```

## 5. Resharding: split または merge

```mermaid
sequenceDiagram
    participant K as Kinesis
    participant C as Parent ShardConsumer
    participant R as Parent RecordProcessor
    participant D as DynamoDB<br/>Lease テーブル
    participant Child as Child ShardConsumer

    K->>K: parent shard を closed にする
    K->>K: child shard を open にして新規データを受信
    Note over K: parent には resharding 前の未処理データが残る場合がある

    K-->>C: parent の最後の batch と end-of-shard
    C->>R: processRecords(last batch)
    R->>D: 最後の通常 checkpoint
    C->>R: shardEnded()
    R->>D: checkpoint(SHARD_END)
    D-->>Child: parent 完了状態を確認
    Child->>K: child shard の読み取り開始
```

## 6. Pod の正常終了

```mermaid
sequenceDiagram
    participant K8s as Kubernetes
    participant App as Spring アプリ
    participant L as SmartLifecycle
    participant S as Scheduler
    participant R as 各 shard の RecordProcessor
    participant D as DynamoDB
    participant SDK as AWS SDK / Netty

    K8s->>App: SIGTERM
    App->>L: Spring shutdown 開始
    L->>S: startGracefulShutdown()

    loop 保持している shard ごと
        S->>R: shutdownRequested()
        R->>D: 最終安全 checkpoint
        S->>D: lease handoff / release
    end

    S-->>L: graceful shutdown Future 完了
    L-->>App: callback.run()
    App->>SDK: Kinesis/DynamoDB client close
    SDK->>SDK: Netty event loop 終了
    App-->>K8s: process 終了
```

## 重要な原則

`checkpoint` は「この sequence までの record が downstream に安全に反映済み」であることを示します。業務用 DynamoDB の更新に失敗した record より先へ checkpoint してはいけません。
