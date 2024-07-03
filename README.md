# GitHub Actions deployment with OIDC

GitHub ActionsとAWS OpenID Connect (OIDC)を使用したデプロイメントプロセスを確認

## CloudFormationスタックのデプロイメント手順

### 前提条件

- AWS CLIがインストールされていること
- AWS CLIが適切に設定されていること（`aws configure`を実行済み）
- 適当なS3バケットが作成されていること

### 手順

以下のAWS CLIコマンドを実行してCloudFormationスタックを作成：

   ```bash
   aws cloudformation create-stack \
     --stack-name github-oidc-stack \
     --template-body file://oidc.yaml \
     --parameters \
       ParameterKey=GitHubOrg,ParameterValue=<GitHubのユーザー名> \
       ParameterKey=RepositoryName,ParameterValue=<リポジトリ名> \
       ParameterKey=S3BucketName,ParameterValue=<アクセスを許可するS3バケット名> \
     --capabilities CAPABILITY_IAM
   ```

   注意: `<>`で囲まれた部分を適切な値に置き換え

スタックの作成状況を確認するためのコマンド：

   ```bash
   aws cloudformation describe-stacks --stack-name github-oidc-stack
   ```

スタックが正常に作成されたら、以下のコマンドで出力を取得：

   ```bash
   aws cloudformation describe-stacks --stack-name github-oidc-stack --query "Stacks[0].Outputs[?OutputKey=='Role'].OutputValue" --output text
   ```

   これにより、作成されたIAMロールのARNが表示される。

## GitHub Actionsについて

GitHubリポジトリの設定で、以下のシークレットを追加：
   - `AWS_ROLE_ARN`: CloudFormationスタックで作成したIAMロールのARN
   - `S3_BUCKET_NAME`: デプロイ先のS3バケット名
   - `AWS_REGION`: リージョン

### GitHub Actionsワークフローの動作説明

以下のタイミングで動作し、指定されたタスクを実行：

1. **トリガー**:
   - `main`ブランチ(./s3bucket配下)へのプッシュがあった時にワークフローが起動。

2. **ジョブの実行**:
   - ワークフローが起動すると、`deploy`ジョブが開始。

3. **ステップの実行**:
   a. **リポジトリのチェックアウト**:
      - `actions/checkout@v3`アクションを使用して、リポジトリの内容をランナーにクローン。
      - これにより、後続のステップでリポジトリのファイルにアクセスできるようになる。

   b. **AWS認証情報の設定**:
      - `aws-actions/configure-aws-credentials@v2`アクションを使用して、AWS認証情報を設定。
      - このステップでは、OIDCを使用してAWSへの認証を行う。
      - `role-to-assume`パラメータに指定されたIAMロール（GitHub Secretsに保存）を引き受ける。

   c. **S3へのデプロイ**:
      - `aws s3 cp`コマンドを使用して、指定されたローカルディレクトリの内容をS3バケットと同期します。
      - このステップでは、リポジトリ内の指定されたパスにあるファイルがS3バケットにアップロードされる。

## 参考
https://blog.serverworks.co.jp/github-actions-oidc
