# コンピュート・データ・メッセージング

## SQS / SNS / Lambda

- SQS は、可視性タイムアウト、DLQ（maxReceiveCount を超えると移動、リドライブで戻せる）。標準キューは、順序の保証がなく、少なくとも 1 回の配信
- 滞留の確認は ApproximateNumberOfMessagesVisible（受信できる件数）。ApproximateAgeOfOldestMessage は最古の経過秒数
- SNS から複数の SQS へのファンアウトで、受け手ごとにバッファでき、並列度も個別に調整できる
- Lambda のイベントソースマッピング（SQS）は、バッチサイズと最大同時実行数を設定できる。ReportBatchItemFailures を有効にして、失敗したメッセージの ID だけを返すと、成功した分は再処理されない
- Kinesis Data Streams ＋ Lambda は、シャードごとに呼び出される。並列度を上げる手段は、シャード数、ParallelizationFactor（最大 10）、拡張ファンアウト。バッチサイズの拡大では上がらない
- Kinesis のエラー処理は、BisectBatchOnFunctionError（失敗したバッチを二分割して絞り込む）、MaximumRetryAttempts、失敗時の送信先
- GetRecords.IteratorAgeMilliseconds が増えるのは、コンシューマーが追いついていないサイン。保持期間の既定は 24 時間（延長できる）
- Kinesis Agent や Firehose で、ログファイルを送れる。Firehose は、バッファして S3 / OpenSearch / Redshift などへ配信する（ネイティブの連携）
- Step Functions は、状態ごとに Retry / Catch、並列（Parallel）、待機（Wait）が使える。Standard は長時間・実行履歴あり、Express は高頻度
- Lambda から RDS / Aurora へ接続するときは、RDS Proxy で接続をまとめ、接続の初期化はハンドラーの外に置いて再利用する。読み取りは、リーダーのエンドポイントに分ける
- Lambda のコールドスタートは、プロビジョニング済み同時実行で避ける。予測できる負荷なら、スケジュールで先回りする

## ECS / EKS / Elastic Beanstalk

- ECS のデプロイコントローラーは、ECS（ローリング）か CodeDeploy（Blue/Green）。ローリングは、サーキットブレーカーで失敗時に戻せる
- Fargate は、タスク実行ロール（イメージの取得、ログの出力）と、タスクロール（アプリの権限）が別
- EKS のコントロールプレーンログは、API、監査、認証などを選んで CloudWatch Logs に出す
- EKS Auto Mode は、ノードのプロビジョニング、パッチ、必須アドオンの管理を AWS に任せる。EBS の CSI ドライバーは IRSA で権限を渡す
- EFS を EKS から使うには、各サブネットにマウントターゲット、SG で TCP 2049、CSI ドライバーに IRSA
- Fargate では DaemonSet が使えないので、ノード前提の仕組みは使えない
- Elastic Beanstalk は、EC2 ベース（Lambda や Fargate は対象外）。デプロイポリシーは、All at once / Rolling / Rolling with additional batch / Immutable / Blue-Green（URL のスワップ）
- Beanstalk 環境の中に RDS を作ると、環境と一緒に消える。残したいなら、環境の外に作る
- サーバーの保守を減らしたいなら、コンテナ化して ECS on Fargate。Kubernetes と Kafka を使い続けたいなら、EKS ＋ MSK

## データとキャッシュ

- Athena は S3 を直接 SQL でクエリする。Glue クローラーでスキーマをカタログ化し、QuickSight は SPICE に取り込んで可視化する
- OpenSearch は、動的マッピングで、項目が変わる JSON ログも検索できる。Serverless もある
- Redshift の同時実行スケーリングは、読み取りが一時的に増えたとき、追加のクラスターが自動で加わる（使った秒数の課金）
- ElastiCache は、セッションなどを外に出して、EC2 をステートレスにする（マルチ AZ、自動フェイルオーバーあり）。DAX は DynamoDB の読み取りキャッシュで、コードの変更が小さい
- Aurora Serverless v2 は、MySQL / PostgreSQL 互換のサーバーレス
- EFS は NFS のみ。FSx for Windows は SMB のみ。FSx for ONTAP は、SMB と NFS を同じボリュームで提供する
- Lambda は EFS をマウントできる。FSx は、Lambda からはマウントできない
- EBS の暗号化は、既存のボリュームにはそのままかけられない。暗号化したスナップショットのコピーから作り直す
- EBS のデフォルト暗号化は、これから作るボリュームに効く設定

## ECS のタスクが止まる理由を調べる

- EventBridge の ECS Task State Change イベント（lastStatus が STOPPED）には、stoppedReason とコンテナの終了コードが含まれる。ルールのターゲットを CloudWatch Logs にすると、Logs Insights で調べられる
- 止まったタスクは、一定時間がたつと API から見えなくなるので、定期的に DescribeTasks で拾う方式は、取りこぼしが出る
