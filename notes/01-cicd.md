# CI/CD（CodePipeline / CodeBuild / CodeDeploy / ECR / Image Builder / CodeArtifact）

## CodePipeline

- 構造は、パイプライン → ステージ → アクション。アクションは、カテゴリ（Source / Build / Test / Deploy / Approval / Invoke）とプロバイダー（実行する相手）で決まる
- 同じステージ内のアクションは runOrder で順序づける（同じ番号は並列）。前のステージが成功したときだけ、次のステージへ進む
- アクションに region を指定すると、別リージョンで実行できる。その場合はリージョンごとにアーティファクトストア（S3）が要り、CodePipeline が自動でコピーする
- 別アカウントへデプロイするときは、アーティファクトストアに KMS のカスタマーマネージドキーを使い、デプロイ先のロールにキーと S3 の許可を出す
- パイプラインタイプは V1 と V2。V2 では、トリガーのフィルター（ブランチ・タグ・ファイルパス）と、実行モード（SUPERSEDED / QUEUED / PARALLEL）が使える
- 変更検知は EventBridge ルールが既定。ルールが欠けると、コミットしても起動しない（作成直後の初回実行は別経路で動く）
- アクションに名前空間を付けると、#{名前空間.変数名} で後続アクションから参照できる。CloudFormation アクションの Outputs も同じ仕組み
- 手動承認アクションは、PutApprovalResult で承認・却下する。承認できる人は IAM で限定する
- Lambda 呼び出しアクションは、PutJobSuccessResult / PutJobFailureResult で結果を返す。長く続く処理は、continuationToken を付けて返すと、再度呼び出される
- Step Functions 呼び出しアクションは、Standard のステートマシンなら完了まで待って、結果をアクションの成否に反映する
- CloudFormation アクションは、変更セットの作成 → 承認 → 変更セットの実行、の順に組むのが定番
- 専用のプロバイダーがないサービスを呼びたいときは、Lambda 呼び出しアクションが汎用の手段になる

## CodeBuild

- buildspec のフェーズは install → pre_build → build → post_build。post_build は build が失敗しても実行される（成否は環境変数 CODEBUILD_BUILD_SUCCEEDING で判定できる）
- artifacts は成果物のファイルをそのまま出力する。reports は JUnit XML・Cobertura・JaCoCo などの対応形式を解析して、テスト結果とカバレッジを集計する
- env.parameter-store / env.secrets-manager で、値を環境変数に注入できる（ログではマスクされる）
- コマンドが非ゼロで終わると、ビルドは失敗し、パイプラインも後続ステージへ進まない
- GitHub などの PR イベントは Webhook（フィルターグループ: PULL_REQUEST_CREATED / PULL_REQUEST_UPDATED など）で起動する。CodeCommit のイベントは EventBridge で受ける
- git tag や git push など Git 操作が要るときは、ソースを完全なクローンで取得する（パイプラインの既定の成果物は、.git のない zip）
- Docker のビルドは特権モード。ECR へのログインは `aws ecr get-login-password | docker login`
- ビルド環境は、独自の Docker イメージ（ECR）にできる

## CodeDeploy

- プラットフォームは、EC2/オンプレ、ECS、Lambda。インプレースがあるのは EC2/オンプレだけで、ECS と Lambda は、トラフィックを切り替える Blue/Green
- 構成は、アプリケーション 1 つに、デプロイグループが複数。デプロイ先、デプロイ設定、アラーム、ロールバックは、デプロイグループで決める
- EC2 の対象指定は、タググループ（グループ間は AND、グループ内の複数タグは OR）か、Auto Scaling グループ。1 つのデプロイグループに複数の ASG は入れられない
- appspec のフック（EC2 インプレース）: ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService。DownloadBundle と Install は予約で、スクリプトを指定できない
- ロードバランサー使用時は、BeforeBlockTraffic / AfterBlockTraffic / BeforeAllowTraffic / AfterAllowTraffic も加わる。BlockTraffic と AllowTraffic は、CodeDeploy が行う予約イベント
- ApplicationStop は、前回リビジョンの appspec を使うので、初回のデプロイでは動かない
- appspec の files は、複数エントリを書ける。permissions は、配置ではなく権限の設定
- デプロイ設定は、EC2 が OneAtATime / HalfAtATime / AllAtOnce、Lambda と ECS が Canary / Linear / AllAtOnce
- CloudWatch アラームをデプロイグループに関連付けると、デプロイ中にアラームが鳴ったとき停止・ロールバックできる（デプロイ完了後は対象外）
- ECS のローリング更新は、CodeDeploy ではなく ECS 標準のデプロイコントローラー。パイプラインでは Amazon ECS アクション（imagedefinitions.json）で実行する
- ECS の Canary・Linear は、新旧のタスクセットの重みを切り替える方式で、タスクを順に入れ替えるローリングとは別物
- 各インスタンスのエージェントが、S3 などからリビジョンを取得する。インスタンスプロファイルの権限と、エンドポイントへの経路が要る
- デプロイ中に Auto Scaling が起動したインスタンスには、最後に成功したリビジョンが配られる（バージョン混在の原因）。対策は、デプロイ中の Launch / Terminate の一時停止
- Blue/Green の切り替え後に、元のインスタンスを残す時間は terminationWaitTimeInMinutes（WaitTimeInMinutes は切り替え前の待機）
- Lambda は、SAM の AutoPublishAlias と DeploymentPreference で、エイリアスの重みを段階的に切り替える
- デプロイログは、インスタンス上の deployment-root 配下に残る。複数台なら、CloudWatch エージェントで集約する

## ECR

- ライフサイクルポリシーで、タグの有無・タグのプレフィックス・保持数（imageCountMoreThan）・経過日数（sinceImagePushed）を条件に、イメージを自動で期限切れにできる
- クロスリージョン／クロスアカウントのレプリケーションは、設定した後にプッシュされたイメージが対象。既存のイメージは別途コピーする
- プルスルーキャッシュは、上流レジストリ（Docker Hub、ECR Public など）を指すルールを作り、初回の取得で ECR 内にコピーする
- スキャンは、基本スキャン（OS パッケージ中心）と、拡張スキャン（Amazon Inspector、言語パッケージも対象、継続スキャン）。結果は EventBridge にも発行される
- NAT のないプライベートサブネットから取得するには、ecr.api と ecr.dkr のインターフェイスエンドポイントと、レイヤーの実体がある S3 のゲートウェイエンドポイントが要る
- Fargate がイメージを取得するときの認証は、タスク実行ロール（タスクロールではない）
- クロスアカウントのプルは、リポジトリポリシーで許可する。aws:PrincipalOrgID を使うと、組織内のアカウントをまとめて許可できる

## EC2 Image Builder

- レシピ（ベースイメージ ＋ コンポーネント）→ パイプライン（スケジュール、テスト）→ 配信設定
- 配信設定でできること: 他のリージョン・アカウントへの配布、起動テンプレートの更新、Parameter Store への AMI ID の書き込み
- CloudFormation の SSM パラメータ型で、Parameter Store の AMI ID を参照すると、スタック更新のたびに最新の値になる
- AMI は RAM では共有できない（AMI の共有許可で他アカウントへ）

## CodeArtifact

- ドメイン（ストレージと認証の単位）の下に、リポジトリを作る。リポジトリは、別のリポジトリを上流（アップストリーム）に指定できる
- 公開レジストリ（npmjs、Maven Central など）への外部接続は、リポジトリに付ける。取得したパッケージは、リポジトリに保存される
- バージョンのステータスは、Published / Unfinished / Unlisted / Archived / Disposed / Deleted。Archived はダウンロード不可でアセットは残り、Disposed はアセットが削除される
- パッケージのオリジン制御（publish と upstream をそれぞれ ALLOW / BLOCK）で、同名パッケージの上流からの取り込みを防げる
- ドメインポリシーで aws:PrincipalOrgID を使うと、組織内のアカウントに許可できる。リポジトリの RAM 共有はできない
- クライアントは `aws codeartifact login` で一時トークンを取得する
- 閉域では、codeartifact.api と codeartifact.repositories のエンドポイントと、S3 のゲートウェイエンドポイントが要る。認可（GetAuthorizationToken、GetRepositoryEndpoint、ReadFromRepository、sts:GetServiceBearerToken）は別に必要

## パイプラインの補足

- 別アカウントにデプロイするパイプライン: デプロイ先のアカウントにロールを作り（信頼ポリシーで元のアカウントを許可）、パイプラインのサービスロールがそのロールを AssumeRole する。アーティファクトの暗号化に使う KMS のカスタマーマネージドキーのキーポリシーでも、そのロールを許可する。aws/s3 のキーはキーポリシーを編集できないので、復号を許可できない
- GitHub の 1 つのリポジトリから環境ごとに展開する: 環境ごとのブランチと、環境ごとのパイプライン（ソースは CodeConnections）を作り、本番のパイプラインには手動承認アクションを入れる。SNS の通知だけでは、パイプラインは止まらない
- Marketplace 由来の AMI が使えなくなったときは、稼働中のインスタンスから AMI を作り直し、その AMI で起動テンプレートの新しいバージョンを作って、ASG を新しいバージョンに更新する

## パイプラインの補足（続き）

- コンポーネントごとに更新頻度や検証が違うなら、パイプラインを分ける。S3 の静的サイトは、S3 デプロイアクションで配置できる（`aws s3 sync` を自作しなくてよい）
- 複数のデプロイグループ（例: 3 つの ASG）に同時に配るときは、アプリケーション 1 つに、デプロイグループを分けて、同じステージ内に CodeDeploy アクションを並列に置く。1 つのデプロイグループに、複数の ASG は入れられない
- EC2・ECS・Lambda を 1 つのパイプラインでデプロイするなら、3 つのプラットフォームすべてに対応している CodeDeploy
- パイプライン間の連携: ECR へのプッシュを EventBridge で検知して、次のパイプラインを起動する。SQS の標準キューは、順序を保証しない
- テストは、デプロイより前に置く（Build ステージの中に、runOrder を後にして足す）。デプロイ後のフックは、配布した後の検証
- デプロイ後に一定時間待って、CloudWatch アラームを評価するゲートは、Step Functions（Wait → DescribeAlarms → Choice → Fail）。Lambda の sleep は、実行時間の上限と、待機中の課金がある
- 実機でのテストは、Device Farm をテストアクションのプロバイダーにして、ビルドしたアプリとテストパッケージを入力にする。ログ・動画・スクリーンショットは、レポートから見る
- FIS（Fault Injection Service）の実験は、専用のアクションがないので、Lambda 呼び出しアクションで開始し、状態を確認してから、PutJobSuccessResult / PutJobFailureResult で返す（開始の API は非同期）
- ECS の Canary: CodeDeploy のデプロイ設定 ECSCanary10Percent5Minutes に、ALB の TargetResponseTime のアラームを関連付けると、劣化したときに、自動でロールバックできる
- Inspector の拡張スキャンの検出結果（EventBridge）で、Image Builder のパイプラインを起動して、イメージを再構築できる
- API Gateway のカナリアリリース: ステージの設定で、URL を変えずに、一部のリクエストだけを新しいデプロイに送れる。切り戻しは、カナリアの削除
- 手動承認は、承認者だけに PutApprovalResult を許可し、承認の履歴は CloudTrail（S3 へ配信）で長期に保存する。パイプラインの実行履歴は、保持期間が限られる
- パイプラインの失敗の通知は、EventBridge（ソース aws.codepipeline）→ SNS。可視化は、CloudWatch ダッシュボード
