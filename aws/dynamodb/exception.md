# DynamoDB 例外一覧

| ステータスコード | 例外名 | 説明 | 代表的な状況 | リトライ | 備考 |
|---|---|---|---|:---:|---|
| 400 | `AccessDeniedException` | DynamoDB へのアクセス権限がない、またはリクエストが拒否された | IAM Role に必要な DynamoDB 権限がない | ❌ | IAM Policy や Role を確認し、必要な権限を付与する |
| 400 | `ConditionalCheckFailedException` | 指定した条件を満たしていない | `version = 3` の場合のみ更新する処理で、現在の version が 4 | ❌ | 条件不一致の原因を確認し、業務上想定内であれば正常系として処理する |
| 400 | `IncompleteSignatureException` | AWS リクエストの署名が正しくない | リクエスト署名に必要な情報が不足している | ❌ | Credential、署名処理、リクエスト生成方法を確認して修正する |
| 400 | `ItemCollectionSizeLimitExceededException` | 同じパーティションキーに属する Item Collection がサイズ上限を超えた | LSI があるテーブルで同一パーティションキーのデータが 10 GB を超えた | ✅ | 一時的にリトライ可能だが、継続する場合はパーティションキーや LSI の設計を見直す |
| 400 | `LimitExceededException` | 同時に実行している管理系処理が多すぎる | テーブルやインデックスを大量に作成・更新・削除している | ✅ | 実行中の管理操作が完了するまで待ち、Backoff 後にリトライする |
| 400 | `MissingAuthenticationTokenException` | 必要な AWS 認証情報がない | Authorization Header がない、または形式が不正 | ❌ | Credential や認証設定を確認し、正しい認証情報を設定する |
| 400 | `ProvisionedThroughputExceededException` | 設定された処理能力を超えた | 設定した RCU / WCU を超える読み書きが発生した | ✅ | Backoff でリトライし、継続する場合は RCU/WCU、アクセス集中、パーティション設計を確認する |
| 400 | `ReplicatedWriteConflictException` | 別リージョンから同じ Item が同時に変更されている | Global Table で同一 Item に複数リージョンから同時に書き込んだ | ✅ | Backoff 後にリトライし、頻発する場合は同一 Item への同時更新設計を見直す |
| 400 | `RequestLimitExceeded` | アカウント単位の処理量上限を超えた | Account-level のスループット上限を超えた | ✅ | Backoff でリトライし、継続する場合は Service Quotas の上限緩和を検討する |
| 400 | `ResourceInUseException` | 対象リソースが別の処理で使用中 | `CREATING` 中のテーブルに別の管理操作を実行した | ❌ | テーブルやインデックスの状態を確認し、処理完了後に改めて実行する |
| 400 | `ResourceNotFoundException` | 指定したテーブルやインデックスが存在しない | テーブル名の誤り、Region の誤り、対象リソースが存在しない | ❌ | テーブル名、Region、デプロイ状況、リソース状態を確認する |
| 400 | `ThrottlingException` | リクエストが多すぎるため DynamoDB に制限された | Hot Partition、On-demand の上限、短時間の大量アクセス | ✅ | Backoff でリトライし、`ThrottlingReason` から Hot Partition や上限超過の原因を特定する |
| 400 | `UnrecognizedClientException` | Access Key または Security Token が無効 | Credential が間違っている、または無効になっている | ✅* | Credential の有効期限や設定を確認する。認証情報が誤っている場合はリトライだけでは解決しない |
| 400 | `ValidationException` | DynamoDB に送ったリクエストの内容が不正 | 必須項目不足、型不一致、不正な Expression、範囲外の値 | ❌ | リクエスト内容を確認し、コードまたは入力値を修正する |
| 500 | `InternalServerError` | DynamoDB 内部でエラーが発生した | AWS 側の一時的な内部障害 | ✅ | Backoff でリトライする。Write は重複実行を防ぐため冪等性を確保する |
| 503 | `ServiceUnavailable` | DynamoDB が一時的に利用できない | AWS 側の一時的なサービス障害 | ✅ | Backoff でリトライし、継続する場合は AWS Health や障害情報を確認する |


## 特に注意が必要な例外

### `ConditionalCheckFailedException`

指定した条件式が成立しなかった場合に発生する。

例えば、現在の `version` が `4` であるにもかかわらず、`version = 3` の場合のみ更新するような条件付き更新を実行すると発生する。

```text
現在: version = 4
条件: version = 3 の場合のみ UPDATE
→ ConditionalCheckFailedException
```

DynamoDB 自体の障害ではないため、同じリクエストをそのままリトライしても基本的には解決しない。

**対処:** 条件不一致の原因を確認する。排他制御や重複登録防止など、業務上想定された条件不一致であれば、エラーではなく正常系として扱うことも検討する。

### `ValidationException`

DynamoDB に送信したリクエスト自体に問題がある場合に発生する。

代表例:

- 必須パラメータが不足している
- Attribute のデータ型が想定と異なる
- `UpdateExpression` や `ConditionExpression` が不正
- 許容されていない値を指定している

同じリクエストを何度リトライしても基本的には成功しない。

**対処:** リトライせず、リクエスト内容、入力データ、Expression などを確認してコードを修正する。

### `ProvisionedThroughputExceededException` / `ThrottlingException`

DynamoDB が処理できる量を超えるリクエストが発生した場合に発生する。

一時的なアクセス集中であればリトライによって成功する可能性がある。ただし、即時に大量のリトライを行うと、さらに負荷を増加させる可能性がある。

**対処:** Exponential Backoff を利用してリトライする。継続的に発生する場合は `ThrottlingReason` や CloudWatch メトリクスを確認し、RCU/WCU、Hot Partition、On-demand の上限、パーティションキー設計などを調査する。

### `InternalServerError`

DynamoDB 内部の一時的な問題などにより HTTP 500 が返された場合に発生する。

特に Write 系 API では注意が必要で、クライアントが 500 を受け取っていても DynamoDB 側では書き込みが完了している可能性がある。

```text
PutItem / UpdateItem
        ↓
DynamoDB 側では書き込み成功
        ↓
レスポンス時に 500
        ↓
アプリケーションは失敗と判断
        ↓
同じ処理をリトライ
        ↓
重複処理の可能性
```

**対処:** Backoff を利用してリトライする。ただし Write 処理では、条件式や一意な処理 ID などを利用し、同じ処理が複数回実行されても問題が起きないように冪等性を確保する。

### `UnrecognizedClientException`

Access Key や Security Token などの認証情報を DynamoDB が認識できない場合に発生する。

AWS 公式ドキュメント上はリトライ可能とされているが、Credential 自体が間違っている、期限切れになっているなどの場合は、同じ認証情報でリトライを繰り返しても解決しない。

**対処:** Credential の有効期限、IAM Role、実行環境の認証設定などを確認する。認証情報に問題がなければ、一時的な問題としてリトライを検討する。

## Batch API の注意点

`BatchGetItem` / `BatchWriteItem` では、リクエスト全体がエラーにならなくても一部の Item が処理されない場合がある。

- `BatchGetItem` : `UnprocessedKeys`
- `BatchWriteItem` : `UnprocessedItems`

この場合はバッチ全体を最初から再実行するのではなく、**未処理の Item のみを Exponential Backoff 後に再実行する**。

## 参考

AWS DynamoDB Developer Guide:  
https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Programming.Errors.html
