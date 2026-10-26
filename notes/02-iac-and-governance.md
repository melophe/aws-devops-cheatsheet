# IaC とガバナンス（CloudFormation / StackSets / CDK / Organizations / Control Tower / Config）

## CloudFormation

- Outputs は、スタックが外に見せる値の宣言。Export を付けると、別スタックから Fn::ImportValue で参照できるが、参照されている間は更新・削除できない
- Fn::ImportValue は Resources セクションなどに書く（Parameters の Default には書けない）。Fn::GetAtt は同一スタック内のみ
- 変更セットで、適用前に変更内容を確認できる
- cfn-signal と CreationPolicy（または WaitCondition）で、リソース作成の完了をシグナルで待つ。UpdatePolicy は更新時のポリシー
- カスタムリソースは、Lambda が処理後に ResponseURL（署名付き URL）へ HTTP PUT で結果を返さないと、スタックが待ち続ける
- AWS::CloudFormation::Init（メタデータ）と cfn-init で、EC2 の構成を宣言する。cfn-hup が変更を検知して再適用する。UserData は初回起動時のみ
- ロールバック失敗（UPDATE_ROLLBACK_FAILED）は、原因を手動で解消してから ContinueUpdateRollback で再開する。解消できないリソースは ResourcesToSkip
- DeletionPolicy / UpdateReplacePolicy で、削除・置換時のリソースの扱い（Retain / Snapshot / Delete）を決める
- ドリフト検出は手動実行（DetectStackDrift）。Config の cloudformation-stack-drift-detection-check ルールで、継続的に評価できる
- Hooks（プロアクティブ）は、リソースの作成前に検証して止める
- 再利用は、モジュール・マクロ・ネストスタックなど。ネストスタックは、親が子をまとめて管理する
- Transform: AWS::CodeDeployBlueGreen と Hooks で、ECS の Blue/Green も CloudFormation で宣言できる
- IaC ジェネレーターで、既存リソースからテンプレート（CDK 用の出力もある）を生成できる
- Git 同期（リポジトリリンク）で、コミットからスタックを自動デプロイできる。CodeConnections の接続はリージョナル
- IAM リソースを含むときは、CAPABILITY_IAM（固定名なら CAPABILITY_NAMED_IAM）が必要
- ネストスタックの子の失敗理由は、DescribeStacks で ParentId を指定して子を抽出し、StackStatusReason を見る

## CDK

- assertions モジュール（Template.fromStack、hasResourceProperties、resourceCountIs）で、合成したテンプレートを単体テストできる
- Aspects で、合成時に各リソースを検査・変更できる
- カスタムコンストラクトで構成パターンを部品化して、社内パッケージで共有する

## StackSets

- サービスマネージド型: Organizations と連携し、OU を指定する。自動デプロイを有効にすると、新規アカウントにも展開される。実行ロールの手作成は不要。管理アカウントは対象にならない
- セルフマネージド型: 管理側と各ターゲット側のロールを自分で作り、アカウント ID を指定する。新規アカウントは手動で追加する
- 配るための仕組みで、禁止や強制をするものではない（禁止・強制は SCP）
- アカウントごとに状態が違うときは、Lambda をバックエンドにしたカスタムリソースで、実行時に API を見て分岐する（Conditions は API を呼べない）

## Organizations と SCP

- SCP は権限の上限を決めるだけで、許可は与えない。実効権限は、SCP と IAM ポリシーの共通部分
- 管理アカウントには効かない
- 既定で FullAWSAccess が付いている。許可リスト方式なら外す。拒否リスト方式なら残して Deny を足す
- よく使う条件キー: aws:RequestedRegion、aws:PrincipalArn、aws:PrincipalOrgID、aws:RequestTag
- OU を作ってアカウントを移すと、その OU の SCP が適用される。新規アカウントも、配置先の OU の SCP を継承する
- ルートにアタッチすると、組織全体（管理アカウントを除く）に効く。一部だけ制限するなら、OU を作ってアタッチする
- SCP は管理アカウントの Organizations に作る（CloudFormation の AWS::Organizations::Policy でも作れる）
- 新規作成したアカウントには、OrganizationAccountAccessRole が自動で作られる。招待で参加したアカウントでは作られないので、手動で作る
- CreateAccount は、管理アカウントでのみ実行できる
- 一括請求は、組織全体で利用量が合算される。コスト配分タグは、管理アカウントで有効化する

## Control Tower

- ランディングゾーン（ログアーカイブ、監査アカウント）と、Account Factory で、アカウントの払い出しとガードレールを自動化する
- コントロールの区分は、必須 / 強く推奨 / 選択的。実現方法の種類は、予防的（SCP）/ 検出的（Config）/ プロアクティブ（CloudFormation Hooks）で、別の軸
- 既存アカウントの登録は、AWSControlTowerExecution ロールの作成と、Account Factory 製品の ProvisionProduct
- AFT: Terraform でアカウントを払い出す。機能フラグで、Enterprise Support への登録などを有効にできる
- Account Factory のブループリント: CloudFormation テンプレートを Service Catalog 製品として登録し、アカウント作成と同時に適用する（ハブアカウントと AWSControlTowerBlueprintAccess ロールが要る）
- CfCT: テンプレートと SCP を、リポジトリの manifest で管理して、OU・アカウントに適用する
- Control Tower が上書きする Config レコーダーの設定は、EventBridge（CreateManagedAccount / UpdateManagedAccount / UpdateLandingZone）で Lambda を起動して、再適用する
- EnabledControl（CloudFormation）で、コントロールを宣言的に管理できる

## AWS Config

- マネージドルールと修復アクション（SSM Automation）を組み合わせると、検出と自動修復を同じ仕組みで行える。既存のリソースも評価対象になる
- 組織ルール／コンフォーマンスパックは、管理アカウントまたは委任管理者から、組織のアカウントに一括で配布する。メンバー側では変更・削除できない
- 組織アグリゲーターで、複数アカウント・複数リージョンの結果を集約する。信頼されたアクセスを有効にし、委任管理者を登録する
- カスタムルールの Lambda の呼び出しが失敗するときは、Lambda 側のリソースベースポリシー（config.amazonaws.com からの InvokeFunction）を確認する
- 評価は設定の記録の後に行われる。リソースの作成そのものを止める仕組みではない（止めるなら SCP や CloudFormation Hooks）
- SCP で Config の停止・削除を拒否すると、検出の土台を守れる

## Service Catalog

- 承認済みの CloudFormation テンプレートを製品として登録し、ポートフォリオにまとめて共有する
- 起動制約（起動に使う IAM ロール）とテンプレート制約で、利用者に広い権限を持たせずに、セルフサービスで提供できる
