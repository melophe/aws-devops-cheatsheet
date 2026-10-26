# セキュリティとアクセス管理（IAM / KMS / シークレット / 組織のセキュリティサービス）

## IAM

- クロスアカウントの AssumeRole は、引き受けられる側のロールの信頼ポリシーと、引き受ける側の IAM ポリシー（sts:AssumeRole）の両方が要る
- リソースベースポリシーを持つサービス（S3、KMS、SQS、Secrets Manager など）は、相手のアカウントのロールを直接許可できる。持たないサービス（ECS、ALB など）は、相手のアカウントのロールを引き受けて操作する
- 権限境界は、実効権限が「IAM ポリシーと権限境界の共通部分」になる
- iam:PassRole は、ロールをサービスに渡す権限。iam:PassedToService 条件で、渡し先を限定できる
- IAM Roles Anywhere は、証明書（トラストアンカー）を使って、AWS の外のサーバーが一時認証情報を取得する仕組み。長期のアクセスキーを置かなくてよい
- EKS の IRSA は、OIDC プロバイダーと、サービスアカウントに紐付けたロール。信頼ポリシーで、そのサービスアカウントを指定する
- EKS は、IAM だけでは認可が完結しない。aws-auth（または アクセスエントリ）で、Kubernetes 側の権限にマッピングする
- S3 の ACL は、無効化（バケット所有者強制）が推奨。authenticated-read は、認証された全 AWS ユーザーに読み取りを許す
- IAM Identity Center は、外部 IdP（Entra ID、AD FS など）を SAML でフェデレーションし、SCIM でユーザー・グループを同期する。権限セットを、グループとアカウントの組み合わせに割り当てる

## KMS

- AWS マネージドキー（aws/s3、aws/secretsmanager、aws/ssm など）は、キーポリシーを編集できない。他のアカウントに復号を許可するには、カスタマーマネージドキー（CMK）が要る
- キーポリシーで許可がないと、IAM ポリシーだけでは使えない（キーポリシーが IAM への委任を許している場合を除く）
- クロスアカウントは、キー側（キーポリシー）と、呼び出し側（IAM ポリシー）の両方で許可する
- エンベロープ暗号化は、データキーでデータを暗号化し、データキー自体を KMS のキーで暗号化する方式

## Secrets Manager と Parameter Store

- Secrets Manager の値の取得には、secretsmanager:GetSecretValue と、暗号化キーの kms:Decrypt の両方が要る
- ネイティブのローテーションがあるのは、RDS / Aurora など。ローテーションは、漏洩後の被害を小さくするもので、アクセスを制限するものではない
- Parameter Store には、スタンダードとアドバンストがある。他のアカウントへの共有は、RAM（アドバンストのみ）。リソースポリシーは使えない
- 共有されたパラメータは、フル ARN で参照する。SecureString の復号には、kms:Decrypt が要る
- RAM は「見せる」仕組みで、暗号化キーは別の持ち物（別途の許可が要る）
- CodeBuild では、buildspec の env.parameter-store で注入できる

## 組織のセキュリティサービス

- GuardDuty / Security Hub / Macie / Inspector は、委任管理者と新規アカウントの自動有効化で、組織全体に展開できる
- Macie は S3 の機密データの検出（自動検出）。GuardDuty は脅威検知。Inspector は脆弱性スキャン（EC2、ECR、Lambda）
- CloudTrail は、組織トレイルで全アカウントを記録できる。ログファイルの整合性検証（ハッシュと署名）と、S3 バケットポリシーで、改ざん・削除を防ぐ
- CloudTrail Lake は、イベントデータストアに SQL でクエリできる。組織のイベントデータストアは、委任管理者が作る
- WAF（L7）は、ALB・CloudFront・API Gateway などに関連付けられる（NLB には付けられない）。Network Firewall は、VPC 単位のトラフィック制御
- IMDSv2 を必須にする設定は、既存のインスタンスでも変更できる。Config のマネージドルール（ec2-imdsv2-check）と修復の組み合わせで、継続的に守れる
- EC2 を調査のために隔離するときは、ルールを一切持たないセキュリティグループに差し替える（インスタンスを止めずに、通信を遮断できる）
- IAM ユーザーの作成を検知して無効化するときは、EventBridge で CreateUser を検知し、アクセスキーは Inactive に、コンソールのパスワードは DeleteLoginProfile で消す
