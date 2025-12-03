# IAMロール連鎖デモ

このデモでは、IAMロール連鎖（Role Chaining）の仕組みを実際に試すことができます。

> **注意**: これは学習・デモ目的のコードです。本番環境での使用は推奨されません。デプロイ後は必ずクリーンアップしてください。

**ロール連鎖の流れ:**
初期プリンシパル → RoleA → RoleB → RoleC

**各ロールの権限:**

- RoleA: BucketAへの読み取り権限
- RoleB: BucketBへの読み取り権限
- RoleC: BucketCへの読み取り権限

## セットアップ手順

### 1. 現在のユーザー/ロールARNを取得する

```bash
aws sts get-caller-identity --query Arn --output text
```

**出力例:**

- IAMユーザーの場合: `arn:aws:iam::123456789012:user/your-name`
- IAMロールの場合: `arn:aws:sts::123456789012:assumed-role/YourRole/session`

**重要:**

- `assumed-role` の場合は、`arn:aws:iam::123456789012:role/YourRole` の形式に変換してください
- 以下の例では `123456789012` を実際のアカウントIDに置き換えてください

**自動変換スクリプト:**

```bash
# ARNを取得して変数に保存
CALLER_ARN=$(aws sts get-caller-identity --query Arn --output text)

# assumed-roleの場合は自動変換
if [[ $CALLER_ARN == *"assumed-role"* ]]; then
  PRINCIPAL_ARN=$(echo $CALLER_ARN | sed 's/:sts:/:iam:/' | sed 's/:assumed-role\//:role\//' | sed 's/\/[^\/]*$//')
else
  PRINCIPAL_ARN=$CALLER_ARN
fi

echo "使用するARN: $PRINCIPAL_ARN"
```

### 2. CloudFormationスタックをデプロイする

```bash
# 手動でARNを指定する場合
aws cloudformation deploy \
  --template-file cfn-role-chain.yaml \
  --stack-name role-chain-demo \
  --parameter-overrides InitialPrincipalArn=arn:aws:iam::123456789012:user/your-name \
  --capabilities CAPABILITY_NAMED_IAM

# または、ステップ1で保存した変数を使用
aws cloudformation deploy \
  --template-file cfn-role-chain.yaml \
  --stack-name role-chain-demo \
  --parameter-overrides InitialPrincipalArn=$PRINCIPAL_ARN \
  --capabilities CAPABILITY_NAMED_IAM
```

### 3. デプロイされたリソースを確認する

```bash
aws cloudformation describe-stacks \
  --stack-name role-chain-demo \
  --query 'Stacks[0].Outputs' \
  --output table
```

## ロール連鎖のテスト

### ステップ1: ロールAを引き受ける

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/RoleA-Demo \
  --role-session-name session-A
```

出力された認証情報（AccessKeyId、SecretAccessKey、SessionToken）を環境変数に設定します：

```bash
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
```

**環境変数に設定する理由**

`aws sts assume-role`コマンドは一時的な認証情報を**JSON形式で出力するだけ**で、AWS CLIは自動的にその認証情報を使用しません。環境変数に設定することで、次のAWS CLIコマンドがそのロールの権限で実行されるようになります。

### ステップ2: ロールBを引き受ける（RoleAの認証情報を使用）

前のステップで設定したRoleAの認証情報を使用して、RoleBを引き受けます：

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/RoleB-Demo \
  --role-session-name session-B
```

同様に、返された認証情報を環境変数に設定します。これにより、次のコマンドはRoleBの権限で実行されます。

### ステップ3: ロールCを引き受ける（RoleBの認証情報を使用）

RoleBの認証情報を使用して、RoleCを引き受けます：

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/RoleC-Demo \
  --role-session-name session-C
```

同様に、返された認証情報を環境変数に設定します。これでロール連鎖が完了し、RoleCの権限で操作できるようになります。

## 各ロールの権限確認

### RoleAでS3アクセスをテスト

RoleAの認証情報を設定した状態で：

```bash
# バケット名を取得
BUCKET_A=$(aws cloudformation describe-stacks \
  --stack-name role-chain-demo \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketAName`].OutputValue' \
  --output text)

BUCKET_B=$(aws cloudformation describe-stacks \
  --stack-name role-chain-demo \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketBName`].OutputValue' \
  --output text)

BUCKET_C=$(aws cloudformation describe-stacks \
  --stack-name role-chain-demo \
  --query 'Stacks[0].Outputs[?OutputKey==`BucketCName`].OutputValue' \
  --output text)

# BucketAへのアクセス（成功するはず）
aws s3 ls s3://$BUCKET_A/

# BucketBへのアクセス（失敗するはず）
aws s3 ls s3://$BUCKET_B/
# エラー: An error occurred (AccessDenied)

# BucketCへのアクセス（失敗するはず）
aws s3 ls s3://$BUCKET_C/
# エラー: An error occurred (AccessDenied)
```

**結果:** RoleAはBucketAのみアクセス可能

### RoleBでS3アクセスをテスト

RoleBの認証情報を設定した状態で：

```bash
# BucketAへのアクセス（失敗するはず）
aws s3 ls s3://$BUCKET_A/
# エラー: An error occurred (AccessDenied)

# BucketBへのアクセス（成功するはず）
aws s3 ls s3://$BUCKET_B/

# BucketCへのアクセス（失敗するはず）
aws s3 ls s3://$BUCKET_C/
# エラー: An error occurred (AccessDenied)
```

**結果:** RoleBはBucketBのみアクセス可能

### RoleCでS3アクセスをテスト

RoleCの認証情報を設定した状態で：

```bash
# BucketAへのアクセス（失敗するはず）
aws s3 ls s3://$BUCKET_A/
# エラー: An error occurred (AccessDenied)

# BucketBへのアクセス（失敗するはず）
aws s3 ls s3://$BUCKET_B/
# エラー: An error occurred (AccessDenied)

# BucketCへのアクセス（成功するはず）
aws s3 ls s3://$BUCKET_C/
```

**結果:** RoleCはBucketCのみアクセス可能

## まとめ

このデモでは、以下を確認できます：

1. **ロール連鎖の動作**: 初期プリンシパル → RoleA → RoleB → RoleC
2. **権限の分離**: 各ロールは特定のバケットのみアクセス可能
3. **最小権限の原則**: 各ロールは必要最小限の権限のみを持つ

## クリーンアップ

```bash
aws cloudformation delete-stack --stack-name role-chain-demo
```
