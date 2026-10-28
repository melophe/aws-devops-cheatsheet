# セキュリティサービスと検知

## 組織全体のセキュリティサービス

- Security Hub: 管理アカウントで信頼されたアクセスを有効にし、委任管理者を指定する。新規アカウントを自動で登録する設定をして、集約リージョンを指定すると、クロスリージョンで集約できる。閲覧は、Identity Center の許可セットを、グループとアカウントに割り当てる
- GuardDuty: 委任管理者と、新規アカウントの自動有効化。VPC フローログ・DNS ログ・CloudTrail のイベントなどから脅威を検出し、結果は EventBridge に流れる。メトリクスフィルターは件数の判定で、挙動の判定ではない
- GuardDuty の Lambda Protection は、Lambda の実行時のネットワークの挙動から、不審な動きを検出する。Inspector の Lambda スキャンは、デプロイ時の静的な検査（依存ライブラリの既知の脆弱性、コードの脆弱性）
- Detective は、CloudTrail、VPC フローログ、（任意で）EKS の監査ログを自動で取り込み、グラフで時系列の相関を調べる。GuardDuty の検出結果の調査に使う
- Firewall Manager: 管理者アカウントを委任し、Config を有効にしたうえで、WAF のポリシー（共通のウェブ ACL、スコープは OU とリソースタイプ）を作ると、新しい ALB や API Gateway にも、自動で関連付けられる。StackSets や Config ルールは、新規リソースへの自動適用には向かない
- Config の組織コンフォーマンスパックで、マネージドルール（例: s3-bucket-ssl-requests-only）を、全アカウント・全リージョンに展開し、SSM Automation の修復まで行える
- Inspector で EC2 をスキャンするには、SSM のマネージドノードである必要がある（SSM Agent、AmazonSSMManagedInstanceCore を含むロール、NAT がなければ ssm / ec2messages / ssmmessages のインターフェイスエンドポイント）
- Inspector の拡張スキャン（ECR）の結果は、EventBridge（ソース aws.inspector2、詳細タイプ Inspector2 Finding）で、重大度 CRITICAL などに絞って通知できる

## Macie と暗号化

- Macie: 独自形式の ID は、カスタムデータ識別子（正規表現）で検出する。マネージドデータ識別子は、事前定義のものだけ
- 公開前に検査するなら、ステージング用のバケットに置いて、Macie の分類ジョブで調べてから、本番に移す
- Macie のコストは、ログ用のバケットを自動検出の対象から外す、日次ジョブに最終更新日時の条件を付けて、前回以降の変更分だけにする、で減らせる
- Macie の検出から、Step Functions で個人情報を編集してアカウント間でコピーする場合、処理するロールに、元のキーの kms:Decrypt と、宛先のキーの kms:GenerateDataKey / kms:Encrypt が要る
- 暗号化した AMI を別のアカウントに共有するには、カスタマーマネージドキーが必要。キーポリシーで相手のアカウントを許可し、相手側が CreateGrant で、Auto Scaling のサービスにリンクされたロール（AWSServiceRoleForAutoScaling）に権限を渡す。aws/ebs のキーポリシーは編集できない

## ログと検知

- ルートユーザーのサインインを、組織全体で 5 分以内に通知: 組織トレイル → CloudWatch Logs → メトリクスフィルター（userIdentity.type が Root、eventName が ConsoleLogin）→ アラーム → SNS。EventBridge でも、ルートのログインを検知できる
- 組織トレイルのログは、S3 に配信して、バケットポリシーと MFA 削除で保護し、ログファイルの整合性検証を有効にする
- S3 の不審なアクセスの調査: サーバーアクセスログは、バケットへのすべてのリクエスト（認証に失敗したものも）の、送信元 IP とオブジェクトキーを記録する。CloudTrail のデータイベントは、オブジェクトレベルの API 呼び出しを記録するが、認証が通らなかったリクエストは、残らないことがある。ログの配信先のバケットには、logging.s3.amazonaws.com を許可するバケットポリシーが要る
- Redshift の監査: 接続ログ、ユーザーログ、ユーザーアクティビティログ（クエリの本文）を有効にして、CloudWatch Logs に出力する。ユーザーアクティビティログは、パラメータ（enable_user_activity_logging）の有効化も要る。ダッシュボードには、ログテーブルのウィジェット（Logs Insights のクエリ）を置ける
- CloudFormation のスタックと実リソースのずれ、Elastic IP の関連付け、IAM ユーザーの作成など、API 呼び出しを起点にした検知は、CloudTrail 経由の EventBridge で即時に行える

## ネットワークの防御

- ALB の送信元の制限: WAF で、IP セットに一致したら Allow、既定のアクションを Block にする（SQL インジェクション対策のルールは維持）。「IP が一致しないものを Block」で既定が Allow のままだと、既定が許可になる。ネットワーク ACL はステートレスで、戻りの通信の設定も要る
- すべてのプロトコルで IP を遮断するなら、Network Firewall（Drop のルール）。DNS Firewall は DNS のみ、WAF は HTTP/HTTPS のみ、セキュリティグループは許可のみ
- API Gateway: リソースポリシーで、aws:SourceVpce を条件にすると、特定の VPC エンドポイント経由のみに許可できる（プライベート API）。aws:PrincipalOrgPaths などの条件でプリンシパルを識別するには、メソッドの認可を AWS_IAM にして、呼び出し側が SigV4 で署名する
- API Gateway の mTLS: カスタムドメインで有効にし、トラストストア（認証局のルート証明書をまとめた PEM）を S3 に置く。パートナーごとのクライアント証明書は、Private CA で発行する。ステージのクライアント証明書は、API Gateway からバックエンドを認証するためのもので、方向が逆
