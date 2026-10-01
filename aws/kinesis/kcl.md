# KCL シーケンス集

## 1. 起動と通常処理

```mermaid
sequenceDiagram
    participant App as Spring アプリ
    participant S as Scheduler
    participant D as DynamoDB<br/>Lease テーブル
    participant C as ShardConsumer
    participant P as RecordsPublisher
    participant R as RecordProcessor
    participant K as Kinesis
    participant B as Downstream System<br/>DB / API / Queue

    App->>S: scheduler.run()
    S->>D: lease を取得・確認
    D-->>S: 担当 shard と checkpoint
    S->>C: shard ごとに consumer を作成
    C->>R: initialize(checkpoint)
    R-->>C: 初期化完了
    C->>P: record 購読を開始
    Note over P,K: shard の record batch を取得し<br/>ShardConsumer に渡す retrieval 層

    loop lease を保持している間
        par lease の維持
            S->>D: lease renew
        and レコード処理
            P->>K: GetRecords
            K-->>P: record batch
            P-->>C: ProcessRecordsInput
            C->>R: processRecords(batch)
            R->>B: アプリケーション処理
            B-->>R: 成功
            R->>D: checkpoint
        end
    end
```

## 2. レコード処理中の一時障害

```mermaid
sequenceDiagram
    participant C as ShardConsumer
    participant R as RecordProcessor
    participant B as Downstream System<br/>DB / API / Queue
    participant D as DynamoDB<br/>Lease テーブル

    C->>R: processRecords(batch)
    R->>B: アプリケーション処理
    B-->>R: timeout / throttling / error

    loop 成功または再試行上限まで
        R->>B: exponential backoff 後に再試行
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

`checkpoint` は「この sequence までの record が downstream system に安全に反映済み」であることを示します。失敗した record より先へ checkpoint してはいけません。

## コンポーネントの役割

| コンポーネント | 事実上の役割 | 保持・操作する主な状態 | 運用上の注意点 |
|---|---|---|---|
| `Scheduler` | KCL worker の中心となる実行ループ。割り当てられた shard ごとに `ShardConsumer` を作成・実行し、停止処理も統括する。 | 現在の shard assignment、`ShardConsumer` の集合 | `scheduler.run()` は一つの worker で一度だけ実行する。終了時は `startGracefulShutdown()` の完了を待つ。 |
| `DynamoDBLeaseCoordinator` | lease の取得、更新、release を管理する。内部で lease renewer、taker、discoverer を起動する。 | worker が現在保持する lease | renew に失敗し続けると lease が期限切れとなり、別 worker に引き継がれる。 |
| DynamoDB Lease Table | shard ごとの owner、checkpoint、lease counter、handoff 状態を永続化する。KCL worker 間で共有される。 | `leaseOwner`、`checkpoint`、`checkpointOwner` など | 業務データ用 DynamoDB テーブルとは役割が異なる。checkpoint はこのテーブルに記録される。 |
| `LeaseAssignmentManager` | KCL 3.x の lease assignment と再均衡を判断する。worker metric と lease 状態を基に、どの worker に lease を割り当てるか決める。 | lease の一覧、worker metric、assignment 計画 | leader が実行する制御系の処理。失敗すると再均衡や再割り当てが遅延する。 |
| `PeriodicShardSyncManager` | Kinesis の shard 構造と lease table を定期的に同期する。split / merge 後の shard を検出し、必要な lease を作成する。 | stream ごとの shard / lease の対応 | 失敗しても既存 shard の処理が直ちに失われるわけではないが、新しい shard の検出が遅れる。 |
| `ShardConsumer` | 一つの shard lease のライフサイクルを管理する。初期化、record 処理、shard 終了、lease 喪失、graceful handoff を状態遷移として扱う。 | shard ごとの lifecycle state、shutdown reason | worker 全体ではなく shard 単位のコンポーネント。 |
| `RecordsPublisher` / `PrefetchRecordsPublisher` | Kinesis から record batch を取得し、`ShardConsumer` に渡す。polling と prefetch queue を担当する。 | shard iterator、取得済み batch、prefetch queue | `GetRecords` 失敗は処理の遅延を意味する。checkpoint の更新とは別の層である。 |
| `ShardRecordProcessor` | アプリケーションが実装する shard ごとの処理ロジック。`initialize`、`processRecords`、終了 callback を受ける。 | アプリケーション固有の処理状態 | 処理の成功確認、idempotency、checkpoint の判断はアプリケーション責務。 |
| `RecordProcessorCheckpointer` | RecordProcessor に checkpoint API を提供し、許可された sequence の範囲を検証して lease table に checkpoint を保存する。 | 最後の checkpoint、checkpoint 可能な最大 sequence | checkpoint は「この位置まで安全に処理済み」という宣言。失敗 record より先に進めてはいけない。 |
| `shutdownRequested()` | lease を正常に handoff する前に、対象 shard の RecordProcessor へ最終 checkpoint の機会を与える callback。 | 対象 shard の checkpointer | Pod 全体の終了を必ず意味しない。再均衡で一つの shard だけ移動する場合にも呼ばれる。 |
| `leaseLost()` | worker が shard の lease を失ったことを通知する callback。 | なし | この時点では ownership が保証されないため checkpoint してはいけない。 |
| `shardEnded()` | Kinesis shard が closed になり、最後の record まで到達したことを通知する callback。 | `SHARD_END` checkpoint | `SHARD_END` checkpoint が成功するまで、child shard の開始が待機する場合がある。 |
| AWS SDK Async Client | Kinesis、DynamoDB、CloudWatch などへの API 呼び出しを実行する client。 | HTTP connection pool、request future | KCL の graceful shutdown 完了前に `close()` してはいけない。 |
| Netty Event Loop | AWS SDK async HTTP client の I/O と callback 実行を担う thread / executor。 | HTTP channel、event loop task queue | `event executor terminated` は event loop が終了済みであることを示す。KCL が API 呼び出し中なら shutdown 順序の問題となる。 |
| Spring `SmartLifecycle` | Spring context の start / stop phase で非同期 component の完了を callback として通知できる lifecycle API。 | running 状態、stop callback | KCL graceful shutdown を完了してから AWS SDK client を破棄するための管理点として使える。 |
| Kubernetes | Pod に SIGTERM を送信し、`terminationGracePeriodSeconds` を過ぎても終了しない process を強制停止する。 | Pod の終了猶予時間 | 猶予時間は KCL の処理中 batch、最終 checkpoint、handoff に必要な時間より長く設定する。 |

## 実行単位と包含関係

KCL を理解するには、「何をデプロイするか」と「何が shard を処理するか」を分ける必要がある。もっとも一般的な Kubernetes 構成では、一つの Pod が一つの JVM process、一つの Spring application、一つの `Scheduler` を持つ。この `Scheduler` が `workerIdentifier` を持って KCL fleet に参加するため、実務上は **1 Pod = 1 KCL worker** と扱える。

ただしこれは推奨される運用上の対応関係であり、概念上の同義語ではない。Pod は Kubernetes の配置・終了単位、`Scheduler` は Java の実行コンポーネント、KCL worker は `workerIdentifier` で識別されて lease に参加する論理的な参加者である。

```mermaid
flowchart TB
    Deploy[Kubernetes Deployment<br/>desired replicas: 3]

    subgraph Cluster[Kubernetes Cluster]
        subgraph PodA[Pod A: deployment / restart unit]
            JVMA[JVM process]
            SpringA[Spring ApplicationContext]
            SchedA[Scheduler Runnable<br/>workerIdentifier = worker-a]
            JVMA --> SpringA --> SchedA
        end
        subgraph PodB[Pod B: deployment / restart unit]
            JVMB[JVM process]
            SpringB[Spring ApplicationContext]
            SchedB[Scheduler Runnable<br/>workerIdentifier = worker-b]
            JVMB --> SpringB --> SchedB
        end
        subgraph PodC[Pod C: deployment / restart unit]
            JVMC[JVM process]
            SpringC[Spring ApplicationContext]
            SchedC[Scheduler Runnable<br/>workerIdentifier = worker-c]
            JVMC --> SpringC --> SchedC
        end
    end

    Deploy --> PodA
    Deploy --> PodB
    Deploy --> PodC

    subgraph KCLFleet[KCL worker fleet: logical participants]
        WA[Worker A]
        WB[Worker B]
        WC[Worker C]
    end

    SchedA -->|represents| WA
    SchedB -->|represents| WB
    SchedC -->|represents| WC

    LeaseTable[(DynamoDB Lease Table)]
    WA --> LeaseTable
    WB --> LeaseTable
    WC --> LeaseTable

    WA --> CA[ShardConsumer x N]
    WB --> CB[ShardConsumer x M]
    WC --> CC[ShardConsumer x 0..K]
    CA --> RP[ShardRecordProcessor]
    CB --> RP
    CC --> RP
```

| 階層 | 概念 | 主な責任 | この文書での見分け方 |
|---|---|---|---|
| 1 | Kubernetes Deployment | Pod の desired replica 数と rollout を管理する。 | 「何個の Pod を維持するか」を決める。 |
| 2 | Pod | コンテナを実行し、SIGTERM と `terminationGracePeriodSeconds` の対象になる。 | Pod 再起動の説明ではこの単位を使う。 |
| 3 | JVM process / Spring application | Java process とその bean・lifecycle を持つアプリケーション実行空間。 | `@PreDestroy`、`SmartLifecycle`、AWS SDK client の管理単位。 |
| 4 | `Scheduler` | KCL の中心 `Runnable`。初期化、lease coordination、`ShardConsumer` の lifecycle、shutdown を統括する。 | `scheduler.run()` と `startGracefulShutdown()` の主体。 |
| 5 | KCL worker | `workerIdentifier` で識別され、lease table を通じて shard を担当する論理的な fleet 参加者。通常は一つの `Scheduler` が一 worker を表す。 | 「どの worker が shard を所有するか」の単位。 |
| 6 | lease | 特定 shard を処理する権利と checkpoint を保持する DynamoDB 上の状態。 | 排他所有と failover の単位。 |
| 7 | `ShardConsumer` | 一つの lease、すなわち一つの shard の KCL lifecycle を実行する。 | shard ごとの `initialize` / `processRecords` / shutdown callback の主体。 |
| 8 | `ShardRecordProcessor` | アプリケーションが実装する業務処理 callback。 | record を downstream に反映し、checkpoint 可否を判断する。 |

### 最も一般的な対応関係

```text
Deployment (replicas = 3)
  └─ Pod A / Pod B / Pod C
       └─ JVM process
            └─ Spring ApplicationContext
                 └─ Scheduler
                      └─ KCL worker (unique workerIdentifier)
                           └─ 0..N leases
                                └─ ShardConsumer per lease
                                     └─ ShardRecordProcessor per shard
```

`workerIdentifier` は同時に動く KCL worker ごとに一意でなければならない。同じ identifier を複数 Pod で共有すると、同じ worker として扱われ、lease 管理が正しく動かない。通常は Pod UID のように再作成時にも重複しない値を使う。

## worker と shard の割当

KCL worker は、`Scheduler.run()` を実行している**一つの KCL 実行インスタンス**である。Kubernetes では通常「一つの Pod 内の一つの application process」が一 worker になるが、概念上 worker = Pod ではない。同一 Pod 内で複数 `Scheduler` を起動すれば複数 worker になり得るが、運用上は一 Pod 一 worker とするのが分かりやすい。

KCL では shard の担当単位は `lease` である。worker は複数 lease を取得できるため、**一つの worker が複数 shard を処理できる**。一方、正常状態では一つの lease は一つの worker だけが所有するため、**同じ shard を二つの worker が同時に正規処理することはない**。

```mermaid
flowchart TB
    LeaseTable[(DynamoDB<br/>Lease Table)]

    subgraph W1[Worker A / Pod A]
        A1[ShardConsumer A-0<br/>RecordProcessor]
        A2[ShardConsumer A-1<br/>RecordProcessor]
        A3[ShardConsumer A-2<br/>RecordProcessor]
    end

    subgraph W2[Worker B / Pod B]
        B1[ShardConsumer B-0<br/>RecordProcessor]
        B2[ShardConsumer B-1<br/>RecordProcessor]
    end

    subgraph W3[Worker C / Pod C]
        C1[ShardConsumer C-0<br/>RecordProcessor]
    end

    LeaseTable -->|lease shard-0| A1
    LeaseTable -->|lease shard-1| A2
    LeaseTable -->|lease shard-2| A3
    LeaseTable -->|lease shard-3| B1
    LeaseTable -->|lease shard-4| B2
    LeaseTable -->|lease shard-5| C1

    A1 --> S0[Kinesis shard-0]
    A2 --> S1[Kinesis shard-1]
    A3 --> S2[Kinesis shard-2]
    B1 --> S3[Kinesis shard-3]
    B2 --> S4[Kinesis shard-4]
    C1 --> S5[Kinesis shard-5]
```

この図では 6 shard を 3 worker に分けている。Worker A は 3 shard、Worker B は 2 shard、Worker C は 1 shard を担当する。実際の割当は worker metric、lease 数、再均衡の進行状況で変化するため、常に完全な均等配分になるとは限らない。

| 状況 | 結果 |
|---|---|
| shard 4 個、worker 1 個 | 1 worker が最大 4 lease を持ち、4 shard を処理できる。 |
| shard 4 個、worker 2 個 | KCL は lease を再均衡し、通常は各 worker が約 2 shard を担当する。 |
| shard 4 個、worker 8 個 | 同時に正規処理できる shard は 4 個まで。残りの worker は lease を持たず idle になり得る。 |
| worker が停止・lease renewal に失敗 | lease の有効期限後、別 worker が lease を取得して最後の checkpoint から再開する。checkpoint 前の処理は重複し得る。 |

## leader worker と lease assignment

KCL 3.x では worker 群の中から leader が一つ選ばれる。`DynamoDBLockBasedLeaderDecider` は DynamoDB の lock を使って leader を選出する。leader が担うのは `LeaseAssignmentManager` による **lease assignment の制御**であり、record を一手に処理する役割ではない。

leader 自身も通常の worker である。leader は自分に割り当てられた shard を処理でき、非 leader worker も自分の shard を同時に処理する。違いは、leader だけが「現在の worker と lease の状態を読み、次の lease 割当を書き込む」ことである。

```mermaid
flowchart TB
    Lock[(DynamoDB leader lock)]
    LeaseTable[(DynamoDB Lease Table)]
    Metrics[(Worker metrics / lease state)]

    subgraph Fleet[KCL worker fleet]
        WA[Worker A<br/>leader]
        WB[Worker B<br/>non-leader]
        WC[Worker C<br/>non-leader]
    end

    Lock -->|leader election| WA
    WA -->|LeaseAssignmentManager<br/>load state and calculate assignment| Metrics
    WA -->|write assignment| LeaseTable
    LeaseTable -->|lease shard-0, shard-1| WA
    LeaseTable -->|lease shard-2, shard-3| WB
    LeaseTable -->|lease shard-4, shard-5| WC

    WA --> PA[process assigned records]
    WB --> PB[process assigned records]
    WC --> PC[process assigned records]
```

| 用語 | 意味 | 誤解しやすい点 |
|---|---|---|
| worker | `Scheduler` を実行し、lease を取得して shard を処理する KCL 実行単位。 | Pod と同義ではない。通常の Kubernetes 運用では一 Pod 一 worker になりやすい。 |
| leader worker | worker の一つ。lease assignment などの制御処理を担当する。 | leader だけが record を処理するわけではない。non-leader も割り当て済み shard を処理する。 |
| `LeaseAssignmentManager` | leader でのみ assignment cycle を実行し、lease をどの worker に割り当てるか判断する。 | record の取得・業務処理を実行するコンポーネントではない。 |
| leader lock | leader が一つだけになるための排他 lock。`DynamoDBLockBasedLeaderDecider` では DynamoDB を使う。 | shard lease とは別の概念。leader lock を持つことと、全 shard lease を持つことは別である。 |

leader が停止したり lock を失ったりしても、すべての record 処理が直ちに停止するわけではない。既存 lease を持つ各 worker は処理を継続できる。一方で新しい assignment や再均衡は次の leader が選出されるまで遅延し得る。新 leader が選出されると、その worker が制御処理を引き継ぐ。

```mermaid
sequenceDiagram
    participant A as Worker A (leader)
    participant B as Worker B
    participant DDB as DynamoDB leader lock
    participant LAM as LeaseAssignmentManager

    A->>DDB: leader lock を取得
    A->>LAM: lease assignment を実行
    Note over A,B: A と B はそれぞれの shard を並行処理
    A-xDDB: Pod 終了または lock 喪失
    Note over B: B は既存 lease の record 処理を継続
    B->>DDB: leader lock を取得
    B->>LAM: assignment を再開
```

## 全体構成とレイヤー

```mermaid
flowchart TB
    K8s[Kubernetes<br/>Pod lifecycle] --> Spring[Spring Application]

    subgraph AppLayer[アプリケーション・ライフサイクル層]
        Spring --> Lifecycle[SmartLifecycle]
        Lifecycle --> Scheduler[Scheduler]
    end

    subgraph ControlPlane[KCL 制御プレーン]
        Scheduler --> Coordinator[DynamoDBLeaseCoordinator]
        Scheduler --> Assignment[LeaseAssignmentManager]
        Scheduler --> ShardSync[PeriodicShardSyncManager]
    end

    subgraph DataPlane[KCL データプレーン: shard ごと]
        Scheduler --> Consumer[ShardConsumer]
        Consumer --> Publisher[RecordsPublisher<br/>PrefetchRecordsPublisher]
        Consumer --> Processor[ShardRecordProcessor]
        Processor --> Checkpointer[RecordProcessorCheckpointer]
        Processor --> Downstream[Downstream System<br/>DB / API / Queue]
    end

    subgraph AwsClients[AWS SDK / Netty]
        SDK[KinesisAsyncClient<br/>DynamoDbAsyncClient] --> Netty[Netty Event Loop]
    end

    Coordinator --> SDK
    Assignment --> SDK
    ShardSync --> SDK
    Publisher --> SDK
    Checkpointer --> SDK

    SDK --> Kinesis[Kinesis Data Streams]
    SDK --> LeaseTable[DynamoDB Lease Table]
```

## shard 処理の内部関係

```mermaid
flowchart LR
    Lease[DynamoDB lease<br/>owner + checkpoint] --> Scheduler[Scheduler]
    Scheduler --> Consumer[ShardConsumer<br/>shard ごとに一つ]
    Consumer --> Publisher[RecordsPublisher]
    Publisher --> Kinesis[Kinesis shard]

    Consumer --> Processor[ShardRecordProcessor]
    Processor --> Business[Downstream System<br/>DB / API / Queue]
    Processor --> Checkpointer[RecordProcessorCheckpointer]
    Checkpointer --> Lease

    Processor -. shutdownRequested .-> Checkpointer
    Processor -. shardEnded .-> Checkpointer
    Processor -. leaseLost: checkpoint 禁止 .-> Lease
```

構成図は「誰が誰を管理・呼び出すか」を、sequence diagram は「いつ何が起きるか」を表します。両方を併用すると、KCL の全体像と障害時の流れを分けて確認できます。

## QA

### Q1. `processRecords()` で例外を投げると、KCL はどのように動作するか？

A. KCL の `ProcessTask` は `ShardRecordProcessor.processRecords()` 呼び出しで発生したアプリケーション例外を捕捉し、失敗した record を示すログを残せる。その例外を「同じ batch を必ず再配信する要求」として扱うわけではない。task の lifecycle は継続し、後続の record batch が `processRecords()` に渡される可能性がある。

したがって、`throw` だけで同一 batch の自動再試行を期待してはいけない。特に危険なのは、失敗した batch の後で別 batch の処理が成功し、より大きい sequence number に checkpoint するケースである。checkpoint はその sequence 以下がすべて安全に処理済みであることを意味するため、未解決の失敗 record まで再開対象から外れてしまう。

```text
checkpoint: 100
batch A: 101-120 -> 110 で処理失敗
batch B: 121-140 -> 処理成功
checkpoint: 140

再起動後: 141 から再開
結果: 110 を含む 101-120 が再取得されない
```

失敗時のアプリケーション側の選択肢は、同じ record または batch を成功するまで再試行する、耐久的な DLQ/inbox に保存してから処理済みとして扱う、または worker を停止して最後の安全な checkpoint から再開する、のいずれかである。失敗をログだけで終わらせ、後続 checkpoint を許可する実装は避ける。

### Q2. record を処理した後に checkpoint を更新しなければ、次の record は処理されないか？

A. 処理される。checkpoint は Kinesis の live な読み取り cursor を止める仕組みではなく、DynamoDB lease table に保存される**永続的な再開位置**である。worker が lease を保持し、`RecordsPublisher` が正常に動作している限り、KCL は shard iterator を進めて後続 record を取得し、`processRecords()` を呼び続けられる。

```text
Kinesis の in-memory iterator: 100 -> 101 -> 102 -> 103 -> ...
DynamoDB checkpoint:          100 のまま
```

この状態で Pod が再起動する、lease を失う、または failover が起きると、新しい worker は DynamoDB にある checkpoint 100 を基準に開始する。そのため 101 以降は再配信され、重複処理が起きる。これは KCL の at-least-once セマンティクスとして正常な挙動である。

一方で、checkpoint を更新しないことは「失敗 record より先へ進んでも安全」という意味ではない。KCL が後続 record を読み続けるため、アプリケーションは安全に処理済みの**連続区間**だけを checkpoint しなければならない。batch 全体が成功した後に checkpoint する、または最後に連続して成功した sequence を明示的に管理する必要がある。

### Q3. KCL が Exactly-once ではなく At-least-once を基本とする技術的・経済的な理由は何か？

A. KCL が管理できるのは主に「Kinesis のどこまで読んだか」という shard ごとの checkpoint であり、アプリケーションが更新する downstream system の状態までは管理できない。end-to-end Exactly-once を実現するには、少なくとも次の三つを一つの原子的な commit として扱う必要がある。

```text
1. source の読み取り位置
2. consumer の内部状態
3. downstream system への副作用
```

Kinesis の checkpoint は DynamoDB lease table、業務処理は任意の DB・API・queue に書き込まれる。この複数システムをまたぐ原子性には、分散 transaction、two-phase commit、または sink 側の transaction / idempotency が必要になる。これらは write latency、可用性、運用複雑性、障害復旧コストを増加させる。

At-least-once は、downstream の成功後に checkpoint を保存するだけで「損失を避ける」設計を実現できる。checkpoint 保存前に障害が起きた場合は record を再配信するため、重複は発生し得るが損失を回避しやすい。Kinesis Data Streams の KCL は at-least-once 処理を前提とする。したがって、重複の排除は KCL ではなく downstream の idempotency で担保するのが基本となる。

### Q4. KCL で重複データが発生する代表的なシナリオは何か？

A. 最も代表的なのは、downstream の処理成功と checkpoint 保存の間に障害が起きるケースである。

```text
record 101 の downstream 処理成功
-> checkpoint 101 を保存する前に Pod crash
-> 新しい worker は checkpoint 100 から開始
-> record 101 を再処理
```

他にも次のケースで重複が起きる。

- DynamoDB lease table への checkpoint 更新が失敗した後、worker が再起動・failover する。
- lease renew の失敗、network partition、OOM、SIGKILL により worker が lease を失う。
- graceful handoff 時に `shutdownRequested()` 内の最終 checkpoint が未実装、失敗、または timeout する。
- downstream request が timeout し、実際には成功したか失敗したか判定できないため、同じ idempotency key で再試行する。
- resharding や deployment のタイミングで、最後の保存済み checkpoint より後の record が再配信される。

重複は異常ではなく recovery の正常動作として扱う。downstream system には event ID、または shard ID と extended sequence number を用いた idempotency key を持たせる。

### Q5. KCL は checkpoint をどこに、どのように記録するか？ checkpoint 更新でエラーが起きるとどうなるか？

A. KCL は DynamoDB lease table の lease item に checkpoint を記録する。checkpoint は shard ごとの sequence number であり、KPL aggregation を使用する場合は sub-sequence 情報も扱う。KCL は現在の worker が lease を保持していることを確認する条件付き更新として checkpoint を保存し、lease ownership を失った worker が古い checkpoint を書き込むことを防ぐ。

checkpoint 更新に失敗した場合、checkpoint は前進しない。次の worker は最後に成功した checkpoint から再開するため、すでに downstream に反映済みの record が再配信される可能性がある。これは at-least-once として期待される動作である。

失敗理由により判断は異なる。

- `ShutdownException`: lease を失った可能性がある。checkpoint を再試行せず、`leaseLost()` として扱う。
- `InvalidStateException`: lease table の状態・設定・アクセスに問題がある。修復されるまで安全な checkpoint はできない。
- `DependencyException`、timeout、throttling: 一時障害の可能性がある。ownership が有効な間だけ再試行し、成功しなければ failover 時の再配信を前提にする。

### Q6. resharding 時に、重複と順序保証で何に注意するか？

A. split または merge が起きると、parent shard は closed になり、新しい child shard は open になって新規 write を受け始める。child shard には、consumer が parent の backlog を読み切る前から record が蓄積し得る。

KCL は parent shard の最後の record を処理し、`shardEnded()` で `SHARD_END` checkpoint を保存してから child shard の処理を開始する。split の child、merge の child はすべての parent の完了を待つため、shard lineage に沿った処理順を維持する。

ただし、これは stream 全体の total order を意味しない。もともと異なる shard 間には total order がなく、Kinesis が保証する順序は shard 内および partition key の系統に限られる。`SHARD_END` checkpoint が失敗すると child 処理が遅延する。resharding 自体は正常動作で重複を必ず発生させるものではないが、parent 完了前後の障害・再試行・checkpoint 失敗では通常どおり重複を考慮する。

### Q7. 重複を絶対に許容できず、Exactly-once に近い処理が必要な場合の選択肢は何か？

A. 任意の external system を含む end-to-end Exactly-once を、KCL だけで提供する選択肢はない。まず、何を一度だけにしたいのかを分けて定義する必要がある。

```text
KCL の record 配信を一度だけにしたい
-> KCL 単体では不可。再配信を前提にする。

業務データへの効果を一度だけにしたい
-> downstream を idempotent / transactional にする。

ストリーム処理エンジン内部の state を一度だけ更新したい
-> Apache Flink の exactly-once checkpointing を検討する。
```

現在のように DynamoDB を downstream に使う場合は、`eventId` または `shardId + extendedSequenceNumber` を処理済みキーとして保存し、業務 item の更新と処理済みキーの登録を DynamoDB `TransactWriteItems` で同時に行う設計が現実的である。KCL の checkpoint は別 transaction になるため再配信自体は起き得るが、業務上の効果を一度だけにできる。

複数 operator を持つ stateful stream processing が必要なら、Amazon Managed Service for Apache Flink または Apache Flink を検討する。Flink は checkpoint により operator state を exactly-once で復元できる。ただし外部 sink まで Exactly-once にするには、source が replay 可能であり、sink が transactional または idempotent でなければならない。Flink の DynamoDB sink は at-least-once であるため、DynamoDB 側の idempotency は引き続き必要になる。

Kafka / Amazon MSK を中心に設計できる場合は、Kafka transaction と `read_committed` consumer を組み合わせた Exactly-once sink を選べる。ただしこれは transaction に参加する Kafka の範囲での保証であり、外部 DB や API まで自動的に Exactly-once になるわけではない。

### Q8. KCL の `耐障害性` とは何か？

A. worker、Pod、process、network の一部が失敗しても、別 worker が shard の処理を引き継ぎ、最後の成功 checkpoint から再開できる性質を指す。KCL は DynamoDB lease table を共有状態として使い、各 worker が保持中の lease を定期的に renew する。

```text
worker A が shard-001 の lease を保持
-> worker A が crash、または lease renew が停止
-> failover time 経過後に lease が期限切れ
-> worker B が shard-001 の lease を取得
-> worker B が DynamoDB の最後の checkpoint から再開
```

この仕組みにより、単一 worker の障害で shard 処理が永続的に停止しにくくなる。ただし、耐障害性は次の意味ではない。

- **Exactly-once ではない。** checkpoint 保存前に処理済みだった record は再配信され得る。
- **データ損失を完全に防ぐものではない。** 失敗 record より先に checkpoint した、Kinesis retention を超えて停止した、または downstream 側でデータを失った場合は KCL だけでは復元できない。
- **無停止を保証するものではない。** failover time、lease 取得、初期化には遅延がある。その間は対象 shard の処理が止まる。
- **すべての障害を自動修復するものではない。** IAM 設定、DynamoDB lease table の削除・破損、Kinesis 側の継続的な障害、アプリケーション bug は別途修復が必要になる。

実務上は、耐障害性を「失敗しても最後の安全な checkpoint に戻り、重複を許容して処理を再開できる設計」と捉える。したがって、checkpoint の正しさ、downstream の idempotency、Kinesis retention、複数 worker の配置が耐障害性の前提となる。

## 参考資料

- [AWS: Kinesis Client Library を使用する](https://docs.aws.amazon.com/streams/latest/dev/kcl.html)
- [AWS: KCL record processor の実装](https://docs.aws.amazon.com/streams/latest/dev/kinesis-record-processor-implementation-app-java.html)
- [AWS: Managed Service for Apache Flink の再起動と checkpoint](https://docs.aws.amazon.com/managed-flink/latest/java/troubleshooting-rt-restarts.html)
- [Apache Flink: Fault Tolerance と Exactly-once](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/)
- [Apache Flink: Source と Sink の Fault Tolerance Guarantees](https://nightlies.apache.org/flink/flink-docs-release-2.3/docs/connectors/datastream/guarantees/)
