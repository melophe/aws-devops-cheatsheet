# IAM の委譲、SCP の評価、Identity Center

## SCP の評価

- ルートから対象アカウントまでの経路上の、すべての階層で許可された操作だけが使える（各階層の許可の共通部分）
- 子の OU に Allow の SCP を足しても、上位の拒否は覆せない。制限を解くには、経路上の SCP を編集する
- 許可リスト方式にしたい OU では、その OU にアタッチされた FullAWSAccess を外し、許可するサービスだけを Allow する SCP をアタッチする。ルートの FullAWSAccess は残してよい（親の許可が前提になるため）
- 拒否リスト方式は、FullAWSAccess を残して Deny を足す。許可リスト方式は、新しいサービスが出ても更新が要らない。拒否リスト方式は、禁止したいものが増えるたびに更新が要る
- メンバーアカウントでは、ルートユーザーを含むすべてのプリンシパルが SCP の影響を受ける。管理アカウントは対象外
- ルートに FullAWSAccess が残っていても、明示的な Deny は優先される。リージョン制限などの Deny 型 SCP は、これで効く。条件付きの Allow 型 SCP は、FullAWSAccess と併存すると制限にならない
- 両方のポリシー（リソース側と ID 側）が許可しているのに拒否されるときは、SCP と権限境界（暗黙の拒否）を確認する

## SCP でよくある書き方

- リージョン制限: aws:RequestedRegion が許可リージョン以外なら Deny（StringNotEquals）。グローバルサービスは除外する。検知は、全リージョンを対象にした組織トレイルと、メトリクスフィルター
- 一部のロールだけ除外: aws:PrincipalARN の StringNotLike 条件
- 送信元の IP を組織全体で強制: aws:SourceIp に対する NotIpAddress で Deny。VPC エンドポイント経由のリクエストには、aws:VpcSourceIp を使う
- S3 のアップロードに KMS 暗号化を強制: s3:x-amz-server-side-encryption が aws:kms でない PutObject を Deny
- CloudTrail を守る: StopLogging / DeleteTrail / UpdateTrail を拒否する
- IAM ユーザーとアクセスキーの作成を禁止: CreateUser / CreateAccessKey / CreateLoginProfile / UpdateLoginProfile を OU で拒否する。アカウント内のすべてのプリンシパルに効き、新しく作られたロールにも効く
- 特定のロール（監査用など）を守る: iam:DeleteRole / UpdateAssumeRolePolicy / AttachRolePolicy / PutRolePolicy などを、そのロールの ARN をリソースにして Deny し、管理用のロールだけを除外する
- Elastic IP の関連付けを禁止: ec2:AssociateAddress を Deny。既存の違反の検知は、Config のルールで継続的に行う
- 一部のアカウントだけに適用: OU を作ってアカウントを移し、その OU に SCP をアタッチする

## 権限の委譲（ロールの作成を開発チームに任せる）

- 権限境界（管理ポリシー）で、委譲先が持てる権限の上限を定義する
- 委譲するポリシーには、iam:PermissionsBoundary 条件キーを付けて、ロール作成時にその境界のアタッチを必須にする。付けないと、境界なしのロールが作れてしまう
- インラインポリシーや管理ポリシーのアタッチ、境界の削除なども、同じ境界を条件にして制限する
- CDK は、bootstrap のオプション（--custom-permissions-boundary）で、作成されるロールに境界を自動で付けられる
- CloudFormation のサービスロール: 信頼ポリシーのプリンシパルを cloudformation.amazonaws.com にして、リソースを操作する権限を持たせる。開発者には、読み取りと CloudFormation の操作、そのサービスロールだけを渡せる iam:PassRole（iam:PassedToService は cloudformation.amazonaws.com）を与え、リソースの直接操作の権限は持たせない
- SCP はアカウントや OU 単位の上限で、ロール単位の境界は付けられない

## Identity Center

- 有効化は、組織の管理アカウントで行う。委任管理者アカウントを登録すると、許可セットの管理を任せられる
- ID ソースは、外部 IdP（SAML 2.0）、Active Directory（AD Connector / Managed AD）、Identity Center のディレクトリ。外部 IdP は SCIM でユーザーとグループを自動で同期する。オンプレの AD から直接 SCIM で同期はできない
- 属性ベースのアクセス制御（ABAC）: アクセス制御の属性を有効にし、IdP の SAML 属性（例: team）を、セッションタグにマッピングする。許可セットのポリシーで、aws:PrincipalTag/team とリソースタグを比較すると、1 つの許可セットで、チーム別に分離できる。チームが増えても、グループの割り当てを足すだけで済む
- IAM ユーザーでのサインインから移行するときは、Identity Center ＋ 外部 IdP で認証を集約し、SCP で IAM ユーザーとアクセスキーの作成を禁止すると、抜け道を塞げる

## 一時認証情報のいろいろ

- EC2 のアプリは、IAM ロール（インスタンスプロファイル）で一時認証情報を使う。長期のアクセスキーを、Secrets Manager や Parameter Store に置くのは、長期キーのまま
- EKS の Pod への権限は、EKS Pod Identity（Pod Identity Agent のアドオンと関連付け、信頼ポリシーのプリンシパルは pods.eks.amazonaws.com）か IRSA。ノードのインスタンスプロファイルにまとめると、同じノードのすべての Pod が同じ権限になる
- AWS の外のデバイス（オンプレの検査機器など）は、IAM Roles Anywhere で、機器ごとの X.509 証明書から一時認証情報を得る。証明書を失効させれば、機器単位で無効にできる
- EFS へは、amazon-efs-utils の TLS マウントで転送中を暗号化し、ファイルシステムポリシーで aws:SecureTransport が false のものを拒否する
- OIDC のみ対応の IdP を使うモバイルアプリは、IAM に OIDC プロバイダーを作成し、aud 条件を付けた信頼ポリシーのロールを作って、アプリが AssumeRoleWithWebIdentity で一時認証情報を得る
- GetFederationToken は、呼び出し元に長期キーが要る

## インスタンスメタデータ（IMDS）

- ec2:RoleDelivery 条件キー（2.0 未満を拒否）の SCP は、IMDSv1 で配られた認証情報での API 呼び出しを止める。稼働中のインスタンスにも、すぐ効く
- ec2:MetadataHttpTokens は、起動時の設定（RunInstances のパラメータ）を縛るもので、稼働中のインスタンスには効かない
- aws:Ec2InstanceSourceVpc と aws:SourceVpc、aws:Ec2InstanceSourcePrivateIPv4 と aws:VpcSourceIp を比較する SCP で、認証情報が発行されたインスタンスの外から使われるのを防げる
