# GitHub Actions から CodeBuild を実行するためのセットアップ手順

## 前提条件
- CodeBuild プロジェクト: `iamapprove` (既に作成済み)
- AWS アカウント ID: `869697965219`
- リージョン: `ap-northeast-1`

## 1. AWS での OIDC プロバイダー設定

### 1.1 IAM で OIDC プロバイダーを作成

1. AWS コンソールで IAM にアクセス
2. 左メニューから「ID プロバイダー」を選択
3. 「プロバイダーを追加」をクリック
4. 以下の情報を入力:
   - **プロバイダーのタイプ**: OpenID Connect
   - **プロバイダーの URL**: `https://token.actions.githubusercontent.com`
   - **対象者**: `sts.amazonaws.com`
5. 「プロバイダーを追加」をクリック

### 1.2 IAM ロールを作成

1. IAM コンソールで「ロール」→「ロールを作成」
2. **信頼されたエンティティタイプ**: 「ウェブアイデンティティ」を選択
3. **ID プロバイダー**: 先ほど作成した `token.actions.githubusercontent.com` を選択
4. **Audience**: `sts.amazonaws.com` を選択
5. 「次へ」をクリック

### 1.3 ポリシーをアタッチ

以下の権限が必要です:
- `AWSCodeBuildDeveloperAccess` (管理ポリシー)

または、カスタムポリシーを作成:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:StartBuild",
        "codebuild:BatchGetBuilds"
      ],
      "Resource": "arn:aws:codebuild:ap-northeast-1:869697965219:project/iamapprove"
    }
  ]
}
```

### 1.4 信頼関係を編集

ロール作成後、「信頼関係」タブで以下のように編集:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::869697965219:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

**重要**: `YOUR_GITHUB_ORG/YOUR_REPO` を実際のリポジトリに置き換えてください。
例: `repo:MynaWallet/codepipeline-iam-approve:*`

### 1.5 ロール ARN をコピー

作成したロールの ARN (例: `arn:aws:iam::869697965219:role/GitHubActionsCodeBuildRole`) をコピーします。

## 2. GitHub での設定

### 2.1 GitHub Secrets を設定

1. GitHub リポジトリの「Settings」→「Secrets and variables」→「Actions」
2. 「New repository secret」をクリック
3. 以下のシークレットを追加:
   - **Name**: `AWS_ROLE_ARN`
   - **Value**: 先ほどコピーしたロール ARN

## 3. 動作確認

1. GitHub リポジトリの「Actions」タブにアクセス
2. 「Run CodeBuild」ワークフローを選択
3. 「Run workflow」をクリック
4. ワークフローが正常に実行されることを確認

## トラブルシューティング

### エラー: "Not authorized to perform sts:AssumeRoleWithWebIdentity"

信頼関係の `token.actions.githubusercontent.com:sub` が正しいリポジトリを指していることを確認してください。

### エラー: "User is not authorized to perform: codebuild:StartBuild"

IAM ロールに適切な CodeBuild 権限がアタッチされていることを確認してください。

## 参考資料

- [AWS: GitHub Actions との OIDC 連携](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [GitHub: OIDC を使用した AWS での認証](https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
