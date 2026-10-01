# Amazon S3 エラー一覧

> AWS 公式ドキュメント「Amazon S3 エラーレスポンス」をもとに、アプリケーション開発で遭遇しやすい主要エラーを整理しています。

| ステータスコード | エラーコード | 説明 | 代表的な状況 | リトライ | 備考 |
|---|---|---|---|:---:|---|
| 400 | `AuthorizationHeaderMalformed` | Authorization ヘッダーの内容が不正 | 署名に使用したリージョンや認証情報がリクエスト先と一致していない | ❌ | リージョン、エンドポイント、署名設定を確認してリクエストを修正する |
| 400 | `BadDigest` | 指定した Content-MD5 / Checksum と S3 が受信したデータの値が一致しない | アップロード中のデータ破損、誤った Checksum を指定した | ❌ | Checksum の計算方法と送信データを確認し、正しい値で再送する |
| 400 | `EntityTooLarge` | アップロードしようとしたデータが許容サイズを超えている | 1 回の PUT で大きすぎるオブジェクトをアップロードした | ❌ | Multipart Upload の利用やアップロード方式を見直す |
| 400 | `ExpiredToken` | 一時的な認証トークンの有効期限が切れている | STS や IAM Role の一時 Credential を長時間使用した | ❌* | Credential を更新してから再実行する。同じ期限切れ Token のままリトライしても解決しない |
| 400 | `IllegalLocationConstraintException` | バケットのリージョンとリクエスト先リージョンが一致していない | `ap-northeast-1` のバケットへ別リージョンのエンドポイントからアクセスした | ❌ | バケットの Region を確認し、正しい Region / Endpoint を使用する |
| 400 | `IncorrectEndpoint` | 対象バケットが別のリージョンに存在する | 間違った S3 Endpoint にリクエストした | ❌ | バケットの Region を確認して正しい Endpoint に送信する |
| 400 | `InvalidArgument` | 指定した引数、ヘッダー、パラメータなどが不正 | 必須 Header 不足、値や形式が不正 | ❌ | AWS の API 仕様とリクエスト内容を確認して修正する |
| 400 | `InvalidPart` | Multipart Upload の Part が存在しない、または ETag が一致しない | CompleteMultipartUpload に誤った Part / ETag を指定した | ❌ | UploadPart の結果と PartNumber / ETag を確認する |
| 400 | `InvalidPartOrder` | Multipart Upload の Part の順序が不正 | CompleteMultipartUpload で PartNumber を昇順に指定していない | ❌ | PartNumber を昇順に並べてリクエストを修正する |
| 400 | `InvalidRequest` | S3 が受け付けられないリクエスト内容になっている | 署名方式、Header、Query Parameter、CopyObject サイズなどが不正 | ❌ | エラーメッセージを確認し、リクエスト内容を修正する |
| 400 | `InvalidSignature` | S3 が計算した署名とリクエストの署名が一致しない | Secret Access Key、署名対象 Header、署名処理などが不正 | ❌ | Credential、SigV4、Region、Header を確認して署名を作り直す |
| 400 | `InvalidToken` | Security Token が無効または形式不正 | 誤った Session Token を使用した | ❌ | Credential を再取得し、正しい Token を使用する |
| 403 | `AccessDenied` | S3 リソースへのアクセスが拒否された | IAM Policy、Bucket Policy、KMS 権限などが不足している | ❌ | IAM、Bucket Policy、Object Ownership、SSE-KMS 利用時は KMS 権限も確認する |
| 403 | `InvalidAccessKeyId` | 指定した Access Key ID が存在しない | 誤った Access Key、削除済みの Credential を使用した | ❌ | 実行環境の Credential を確認し、正しい認証情報に更新する |
| 404 | `NoSuchBucket` | 指定したバケットが存在しない | バケット名の誤り、削除済みバケットへのアクセス | ❌ | バケット名、AWS Account、Region、デプロイ状況を確認する |
| 404 | `NoSuchKey` | 指定したオブジェクトキーが存在しない | Key のタイプミス、削除済みオブジェクトへの GET | ❌ | Bucket / Key を確認する。業務上「データなし」が正常なら正常系として扱う |
| 404 | `NoSuchUpload` | 指定した Multipart Upload が存在しない | Upload ID が無効、既に Complete / Abort 済み | ❌ | Upload ID とアップロード状態を確認し、必要なら Multipart Upload を最初から開始する |
| 409 | `BucketAlreadyExists` | 指定したバケット名が既に使用されている | 他の AWS Account を含め、同じバケット名が既に存在する | ❌ | グローバルで一意になる別のバケット名を使用する |
| 409 | `BucketAlreadyOwnedByYou` | 作成しようとしたバケットを自分が既に所有している | 同じ名前で CreateBucket を再実行した | ❌ | 既存バケットを利用する。意図した処理か確認する |
| 409 | `BucketNotEmpty` | 空ではないバケットを削除しようとした | Object / Version / Delete Marker が残っている | ❌ | バケット内の Object や Version を削除してから再実行する |
| 409 | `ConditionalRequestConflict` | 同じリソースに対する競合する操作が発生した | 条件付き PutObject と別の更新・削除処理が競合した | ✅* | PutObject は再試行可能。Multipart Upload は新しい Upload ID で最初からやり直す |
| 411 | `MissingContentLength` | Content-Length が必要だが指定されていない | HTTP リクエストに Content-Length がない | ❌ | Content-Length を正しく設定してリクエストを修正する |
| 416 | `InvalidRange` | 指定した Range を満たせない | オブジェクトサイズを超える byte range を GET した | ❌ | オブジェクトサイズと Range 指定を確認する |
| 500 | `InternalError` | S3 内部で一時的なエラーが発生した | AWS 側の一時的な内部障害 | ✅ | Exponential Backoff でリトライする。Write は結果が不明な場合があるため冪等性も考慮する |
| 501 | `NotImplemented` | 指定した機能や Header が S3 でサポートされていない | 対象 API / Endpoint で未対応の Header を使用した | ❌ | API、Endpoint、使用 Header の仕様を確認する |
| 503 | `SlowDown` | リクエスト頻度が高く、一時的に処理を抑制されている | 短時間に大量の S3 リクエストを送信した | ✅ | Exponential Backoff + Jitter でリトライし、継続する場合はアクセスパターンを確認する |
| 503 | `ServiceUnavailable` | S3 が一時的にリクエストを処理できない | AWS 側の一時的なサービス障害 | ✅ | Exponential Backoff でリトライする |
| 400 / 408系 | `RequestTimeout` | S3 がリクエスト全体を待機中にタイムアウトした | ネットワーク遅延、アップロード中断、通信が遅すぎる | ✅ | 接続状態を確認し、Backoff 後にリクエストを再実行する |

## リトライ判断の基本

- **リトライ不可（❌）**  
  権限、認証情報、バケット名、オブジェクトキー、リクエストパラメータなどに問題があるケース。同じリクエストをそのまま繰り返しても基本的に解決しない。

- **リトライ可能（✅）**  
  S3 側の一時的な障害、`SlowDown`、通信タイムアウトなど、時間を置くことで成功する可能性があるケース。基本的に **Exponential Backoff + Jitter** を利用する。

- **条件付きリトライ（✅*）**  
  Credential の更新や Multipart Upload の再開始など、単純に同じリクエストを再送するのではなく、状態を確認・更新してから再実行する必要があるケース。

## 特に注意が必要なエラー

### `AccessDenied`

S3 へのアクセス権限が不足している場合に発生する。

単純に IAM Policy だけが原因とは限らない。

代表的な確認箇所:

- IAM Policy
- Bucket Policy
- Service Control Policy (SCP)
- VPC Endpoint Policy
- Object Ownership / ACL
- SSE-KMS 利用時の KMS Key Policy / IAM 権限

**対処:** 同じリクエストをリトライするのではなく、どのポリシーで拒否されているかを確認して権限設定を修正する。

### `NoSuchKey` / `NoSuchBucket`

指定した Object または Bucket が存在しない場合に発生する。

```text
GET s3://example-bucket/data/test.json
                         ↓
               test.json が存在しない
                         ↓
                    NoSuchKey
```

必ずしもシステム障害とは限らない。例えば「ファイルが存在しない場合は処理しない」という仕様なら、`NoSuchKey` は想定内の結果として扱える。

**対処:** Bucket 名、Object Key、Region、生成・削除タイミングを確認する。業務上「存在しない」が正常なケースでは、エラーとして無限リトライしない。

### `SlowDown` / `ServiceUnavailable`

S3 が一時的にリクエストを処理できない場合に発生する。

`SlowDown` は特に、大量のリクエストなどによって一時的に S3 側から処理を抑制された状態。

```text
Request
  ↓
503 SlowDown
  ↓
少し待つ
  ↓
Retry
  ↓
失敗したらさらに待つ
  ↓
Retry
```

**対処:** Exponential Backoff + Jitter でリトライする。AWS SDK の標準 Retry 機能を利用することを基本とし、継続的に発生する場合はリクエスト量やアクセスパターンを調査する。

### `InternalError`

S3 内部で一時的な問題が発生した場合に返される。

基本的にはリトライ対象だが、PUT などの Write 処理では注意が必要。

```text
PutObject
    ↓
S3 側では保存成功
    ↓
レスポンス処理でエラー
    ↓
アプリケーションは失敗と判断
    ↓
PutObject を再実行
```

クライアント側からは「書き込みが成功したか」が分からないケースを考慮する必要がある。

**対処:** Backoff でリトライする。Write 処理では、同じ Key への再 PUT が業務上問題ないかを確認し、必要に応じて条件付きリクエストや処理 ID などで冪等性を確保する。

### `RequestTimeout`

通信途中で S3 がリクエストを最後まで受信できなかった場合などに発生する。

大きなファイルのアップロードでは、ネットワーク切断や通信遅延によって発生する可能性がある。

**対処:** Backoff 後にリトライする。大容量ファイルでは Multipart Upload を利用し、失敗した Part だけを再送できる構成にする。

### `ConditionalRequestConflict`

条件付きリクエストと別の操作が同時に実行され、競合した場合に発生する。

AWS 公式ドキュメントでは、`PutObject` の場合はリクエストを再試行できる。

ただし Multipart Upload の場合は扱いが異なる。

**対処:**

- `PutObject` → 状態を確認した上でリトライ
- Multipart Upload → 新しい `CreateMultipartUpload` を実行し、新しい Upload ID で各 Part を再アップロードする

### `InvalidSignature` / `ExpiredToken`

認証情報や署名に問題がある場合に発生する。

`ExpiredToken` の場合、期限切れの Token で何度リトライしても成功しない。

**対処:** Credential Provider から新しい Credential を取得する。`InvalidSignature` の場合は Access Key / Secret Key だけでなく、Region、Endpoint、SigV4、署名対象 Header、システム時刻なども確認する。

### Multipart Upload 関連

Multipart Upload では通常の `PutObject` より状態管理が重要になる。

主なエラー:

| エラー | 意味 | 対処 |
|---|---|---|
| `NoSuchUpload` | Upload ID が存在しない、または既に Complete / Abort 済み | Upload 状態を確認し、必要なら新しい Multipart Upload を開始する |
| `InvalidPart` | 指定した Part または ETag が正しくない | UploadPart の結果として取得した PartNumber / ETag を確認する |
| `InvalidPartOrder` | PartNumber の順序が不正 | PartNumber を昇順に並べて CompleteMultipartUpload を実行する |
| `ConditionalRequestConflict` | 他の処理と競合した | 新しい Multipart Upload を開始し、各 Part を再アップロードする |

Multipart Upload では、単純に同じ `CompleteMultipartUpload` を何度も繰り返すのではなく、**Upload ID、PartNumber、ETag、現在の Upload 状態を確認してからリトライ方法を判断する**。

## 参考

AWS Amazon S3 Developer Guide:  
https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/developerguide/ErrorResponses.html
