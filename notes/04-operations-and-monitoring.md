# 運用と監視（Systems Manager / CloudWatch / EventBridge / Auto Scaling）

## Systems Manager

- マネージドノードにするには、SSM Agent、インスタンスプロファイル（AmazonSSMManagedInstanceCore）、SSM への経路（インターネット、または VPC エンドポイント）が要る。インスタンスプロファイルを付けられないときは、Default Host Management Configuration（DHMC）が使える
- オンプレのサーバーは、ハイブリッドアクティベーションで登録する（ID は mi- で始まる）
- Patch Manager は、パッチベースライン → パッチグループ → メンテナンスウィンドウ（AWS-RunPatchBaseline）の流れ。Scan は確認のみ、Install で適用する
- 1 回のパッチ適用で有効なベースラインは 1 つ。社内リポジトリなどは、代替パッチソースとして、1 つのベースラインに定義する
- State Manager の関連付けは、ドキュメントをスケジュールで継続的に適用する。タグや「すべてのノード」を指定すると、新しく起動したインスタンスにも自動で効く。Run Command は単発の実行
- Automation は、ランブックで複数ステップの作業を自動化する（Config の修復アクションにも使う）
- Inventory は、インストール済みアプリケーションなどを収集する。Config のマネージドルールと組み合わせると、禁止ソフトの検出ができる
- EC2 Instance Connect は、SendSSHPublicKey で、60 秒だけ有効な鍵を送る。Session Manager は、SSH のポートを開けずに接続する
- ドメイン参加は、AWS-JoinDirectoryServiceDomain の関連付け（インスタンスプロファイルに AmazonSSMDirectoryServiceAccess が要る）
- EKS ノードがクラスターに参加しない原因の診断には、Automation の AWSSupport-TroubleshootEKSWorkerNode

## CloudWatch Logs

- メトリクスフィルター: ログの文字列・JSON パターンに一致した件数などを、カスタムメトリクスにする。作成後に届くログが対象で、アラームを設定できる
- サブスクリプションフィルター: ログをリアルタイムに Kinesis / Firehose / Lambda へ転送する。送信先に SNS は選べない。クロスアカウントは、宛先（Destination）を作る。これも作成後のログが対象
- Logs Insights: 保存済みのログを遡ってクエリする（filter、stats）。スキャン量で課金される。アラームを直接設定できない
- Contributor Insights: ログの特定のキーについて、上位の貢献者を集計する
- ログ量の調査は、AWS/Logs 名前空間の IncomingBytes を、LogGroupName のディメンションで見る
- VPC フローログを CloudWatch Logs に送り、宛先ポートと ACCEPT に一致するメトリクスフィルターを作ると、特定ポートへの接続数を数えられる
- WAF のログを CloudWatch Logs に送り、action が BLOCK のものを数えると、ブロックの急増を検知できる
- ログ本文の文字列（例: CRITICAL）で通知するときは、メトリクスフィルター → アラーム → SNS

## CloudWatch メトリクスとアラーム

- アラームは、期間・評価期間・アラームにするデータポイント数（M of N）と、欠損データの処理（notBreaching / breaching / ignore / missing）を指定する
- 異常検出アラームは、学習した通常範囲（バンド）からの逸脱を検知する。傾向が変動するメトリクスに向く
- EC2 アクションの recover は、StatusCheckFailed_System（AWS 側の障害）で使い、インスタンス ID、IP、EBS が保持される。StatusCheckFailed_Instance は OS 側の問題で、再起動で対処する
- CloudWatch エージェントは、メモリ・ディスクなど OS 内の指標、ログファイル、StatsD、Prometheus のスクレイプ、procstat（プロセス単位）を扱える。詳細モニタリングには、メモリやディスクの指標は含まれない
- Container Insights は、ECS / EKS のメトリクス。EKS では、エージェントを DaemonSet で配置し、ノードのロールに CloudWatchAgentServerPolicy が要る。Pod 単位のメモリは pod_memory_utilization
- メトリクスストリームは、メトリクスを Firehose 経由で継続的に配信する。フィルターなしなら、新規のメトリクスも自動で含まれる
- X-Ray は分散トレース。X-Ray デーモンは、バッファして非同期で送るので、応答時間に影響しない。API Gateway はアクティブトレースを有効にする
- DevOps Guru は、機械学習で運用データを分析し、異常と推奨を提示する。SNS の通知チャネルを設定する
- 予定の再起動やインスタンス退役は、AWS Health のイベントで事前にわかる

## EventBridge

- AWS API の呼び出しは、CloudTrail 経由のイベント（detail-type: AWS API Call via CloudTrail）として検知できる
- クロスアカウントは、受信側のイベントバスにリソースベースポリシー（events:PutEvents の許可）を付け、送信側のルールのターゲットを相手のバスにする。ポリシーは受信側、ルールは送信側
- ルールのターゲットには、events.amazonaws.com が引き受けるロールを指定する。届かないときは、イベントのパターン、メトリクスの TriggeredRules と FailedInvocations、ロールの権限の順で確認する
- ターゲットにデッドレターキューを設定すると、配信に失敗したイベントを保全できる。CloudWatch Logs をターゲットに足すと、バスに届いたイベントの本文を確認できる
- CodePipeline の通知は、ソース aws.codepipeline のイベント（Pipeline / Stage / Action の状態変化）で絞れる。独自の形式の Webhook へ送るなら、Lambda で整形する

## Auto Scaling

- ターゲット追跡は、指標を目標値に保つ。SQS のバックログ（キュー長 ÷ タスク数）のように、カスタムメトリクスも使える。ステップスケーリングは、アラームの段階に応じて増減する。スケジュールスケーリングは、予測できるパターンに先回りする
- ウォームプールは、初期化済みのインスタンスを待機させる（Stopped は EBS の料金のみ、Running は通常の料金）。ライフサイクルフックで、初期化の完了を待てる
- ライフサイクルフックは、Pending:Wait や Terminating:Wait で処理を挟む。完了は complete-lifecycle-action で通知する。タイムアウトの既定は 1 時間
- ヘルスチェックのタイプを ELB にすると、アプリケーション層の異常も、置換の対象になる
- 起動の失敗は、通知タイプ autoscaling:EC2_INSTANCE_LAUNCH_ERROR で直接通知できる
- Lambda の同時実行: プロビジョニング済み（コールドスタート対策。Application Auto Scaling でスケジュールも可）と、予約済み（上限の確保で、初期化遅延は解消しない）
- ECS サービスの healthCheckGracePeriodSeconds は、起動直後に ALB のヘルスチェックで止められないための猶予
