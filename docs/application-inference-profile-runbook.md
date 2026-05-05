# Application Inference Profile 作成・動作確認手順

`scripts/cfn/application-inference-profile.yaml` を使って Amazon Bedrock の Application Inference Profile を作成し、`aws bedrock-runtime converse` で動作確認する手順です。

## 対象

- リージョン: `ap-northeast-1`
- CloudFormation スタック名: `application-inference-profile`
- Application Inference Profile
  - `application-inference-profile-apne1-claude-haiku-45`
  - `application-inference-profile-apne1-nova-2-lite`

## 前提

- AWS CLI が利用できること
- `jq` コマンドが利用できること
- AWS 認証情報が設定済みであること
- Bedrock 利用権限があること
- `ap-northeast-1` で対象モデルを利用できること

## テンプレート

- パス: `scripts/cfn/application-inference-profile.yaml`
- 作成対象:
  - Claude Haiku 4.5 ベースの Application Inference Profile
  - Nova 2 Lite ベースの Application Inference Profile

現在の既定値:

| パラメータ | 値 |
|---|---|
| `InferenceProfileName` | `application-inference-profile-apne1-claude-haiku-45` |
| `ModelSourceProfileId` | `jp.anthropic.claude-haiku-4-5-20251001-v1:0` |
| `Nova2LiteInferenceProfileName` | `application-inference-profile-apne1-nova-2-lite` |
| `Nova2LiteModelSourceProfileId` | `jp.amazon.nova-2-lite-v1:0` |

テンプレートでは、コピー元 ARN をファイルに直書きせず、`AWS::Partition` / `AWS::Region` / `AWS::AccountId` で動的に組み立てます。

## 1. AWS CLI と認証状態の確認

```bash
aws --version
aws configure get region
aws sts get-caller-identity
```

実行例:

```json
{
  "UserId": "<user-id>",
  "Account": "<account-id>",
  "Arn": "arn:aws:iam::<account-id>:user/<user-name>"
}
```

## 2. テンプレートの検証

```bash
aws cloudformation validate-template \
  --template-body file://scripts/cfn/application-inference-profile.yaml
```

返り値にはテンプレート Description と Parameters が含まれます。

## 3. Application Inference Profile の作成・更新

```bash
aws cloudformation deploy \
  --stack-name application-inference-profile \
  --template-file scripts/cfn/application-inference-profile.yaml \
  --parameter-overrides \
    InferenceProfileName=application-inference-profile-apne1-claude-haiku-45 \
    Nova2LiteInferenceProfileName=application-inference-profile-apne1-nova-2-lite \
  --no-fail-on-empty-changeset
```

成功時の例:

```text
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - application-inference-profile
```

## 4. スタック出力の確認

```bash
aws cloudformation describe-stacks \
  --stack-name application-inference-profile \
  --query 'Stacks[0].Outputs' \
  --output table
```

確認できる主な Output:

- `InferenceProfileArn`
- `InferenceProfileIdentifier`
- `InferenceProfileStatus`
- `Nova2LiteInferenceProfileArn`
- `Nova2LiteInferenceProfileIdentifier`
- `Nova2LiteInferenceProfileStatus`

実行例:

```text
---------------------------------------------------------------------------------------------------------------
|                                         DescribeStacks                                                        |
+--------------------------------------+-----------------------------------------------------------------------+
| OutputKey                            | OutputValue                                                           |
+--------------------------------------+-----------------------------------------------------------------------+
| InferenceProfileArn                  | arn:aws:bedrock:ap-northeast-1:<account-id>:application-inference... |
| InferenceProfileStatus               | ACTIVE                                                                |
| Nova2LiteInferenceProfileArn         | arn:aws:bedrock:ap-northeast-1:<account-id>:application-inference... |
| Nova2LiteInferenceProfileStatus      | ACTIVE                                                                |
+--------------------------------------+-----------------------------------------------------------------------+
```

## 5. `aws bedrock` と `jq` で Application Inference Profile 情報を直接取得

CloudFormation の Output を使わず、Bedrock API から直接 ARN / ID / Status を取得する方法です。

### Claude Haiku 4.5

```bash
aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-claude-haiku-45")
  | {
      inferenceProfileName,
      inferenceProfileArn,
      inferenceProfileId,
      status
    }
'
```

ARN だけを変数に取り出す場合:

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-claude-haiku-45")
  | .inferenceProfileArn
')
```

### Nova 2 Lite

```bash
aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-nova-2-lite")
  | {
      inferenceProfileName,
      inferenceProfileArn,
      inferenceProfileId,
      status
    }
'
```

ARN だけを変数に取り出す場合:

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-nova-2-lite")
  | .inferenceProfileArn
')
```

## 6. 作成したプロファイルの詳細確認

### Claude Haiku 4.5

```bash
PROFILE_ARN=$(aws cloudformation describe-stacks \
  --stack-name application-inference-profile \
  --query "Stacks[0].Outputs[?OutputKey=='InferenceProfileArn'].OutputValue | [0]" \
  --output text)

aws bedrock get-inference-profile \
  --inference-profile-identifier "$PROFILE_ARN"
```

`aws bedrock` と `jq` だけで ARN を取得してから詳細を確認する場合:

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-claude-haiku-45")
  | .inferenceProfileArn
')

aws bedrock get-inference-profile \
  --region ap-northeast-1 \
  --inference-profile-identifier "$PROFILE_ARN"
```

返り値例:

```json
{
  "inferenceProfileName": "application-inference-profile-apne1-claude-haiku-45",
  "inferenceProfileArn": "arn:aws:bedrock:ap-northeast-1:<account-id>:application-inference-profile/<profile-id>",
  "inferenceProfileId": "<profile-id>",
  "status": "ACTIVE",
  "type": "APPLICATION"
}
```

### Nova 2 Lite

```bash
PROFILE_ARN=$(aws cloudformation describe-stacks \
  --stack-name application-inference-profile \
  --query "Stacks[0].Outputs[?OutputKey=='Nova2LiteInferenceProfileArn'].OutputValue | [0]" \
  --output text)

aws bedrock get-inference-profile \
  --inference-profile-identifier "$PROFILE_ARN"
```

`aws bedrock` と `jq` だけで ARN を取得してから詳細を確認する場合:

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-nova-2-lite")
  | .inferenceProfileArn
')

aws bedrock get-inference-profile \
  --region ap-northeast-1 \
  --inference-profile-identifier "$PROFILE_ARN"
```

返り値例:

```json
{
  "inferenceProfileName": "application-inference-profile-apne1-nova-2-lite",
  "inferenceProfileArn": "arn:aws:bedrock:ap-northeast-1:<account-id>:application-inference-profile/<profile-id>",
  "inferenceProfileId": "<profile-id>",
  "status": "ACTIVE",
  "type": "APPLICATION",
  "models": [
    {
      "modelArn": "arn:aws:bedrock:ap-northeast-3::foundation-model/amazon.nova-2-lite-v1:0"
    },
    {
      "modelArn": "arn:aws:bedrock:ap-northeast-1::foundation-model/amazon.nova-2-lite-v1:0"
    }
  ]
}
```

## 7. 作成したプロファイルのタグ確認

Application Inference Profile に設定されたタグは、`aws bedrock list-tags-for-resource` で確認できます。

### Claude Haiku 4.5

まず ARN を取得します。

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-claude-haiku-45")
  | .inferenceProfileArn
')
```

全タグを確認する場合:

```bash
aws bedrock list-tags-for-resource \
  --region ap-northeast-1 \
  --resource-arn "$PROFILE_ARN" \
  --output json \
| jq
```

`Owner` タグの値だけ確認する場合:

```bash
aws bedrock list-tags-for-resource \
  --region ap-northeast-1 \
  --resource-arn "$PROFILE_ARN" \
  --output json \
| jq -r '.tags[] | select(.key == "Owner") | .value'
```

### Nova 2 Lite

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-nova-2-lite")
  | .inferenceProfileArn
')
```

全タグを確認する場合:

```bash
aws bedrock list-tags-for-resource \
  --region ap-northeast-1 \
  --resource-arn "$PROFILE_ARN" \
  --output json \
| jq
```

`Owner` タグの値だけ確認する場合:

```bash
aws bedrock list-tags-for-resource \
  --region ap-northeast-1 \
  --resource-arn "$PROFILE_ARN" \
  --output json \
| jq -r '.tags[] | select(.key == "Owner") | .value'
```

返り値例:

```json
{
  "tags": [
    {
      "key": "Name",
      "value": "application-inference-profile-apne1-nova-2-lite"
    },
    {
      "key": "Owner",
      "value": "setup0"
    }
  ]
}
```

## 8. Cost Explorer でコストタグとして使うための有効化

Application Inference Profile に `Owner` などのタグを設定しただけでは、Cost Explorer のタグフィルタにはすぐ出ません。Billing 側で **cost allocation tag** として有効化する必要があります。

有効化手順:

1. AWS Billing and Cost Management コンソールを開く
2. **Cost allocation tags** を選択する
3. タグキー `Owner` を選択する
4. **Activate** を実行する

注意点:

- タグキーが **Cost allocation tags** ページに表示されるまで、タグ設定後 **最大 24 時間**かかることがあります
- 表示後、タグキーの有効化完了まで **最大 24 時間**かかることがあります
- Cost Explorer などのコストデータ更新も日次のため、実際に見え始めるまでさらに時間差があります
- AWS Organizations 配下では、通常 **management account** 側で有効化を行います

参考資料:

- AWS Billing and Cost Management User Guide: [Activating tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/activating-tags.html)
  - user-defined tag をリソースへ付与した後、**Cost allocation tags** ページに表示されるまで **最大 24 時間**
  - その後、タグキーの有効化にも **最大 24 時間**
- AWS Billing and Cost Management User Guide: [Backfill cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-allocation-backfill.html)
  - Backfill 後も、Cost Explorer / Data Exports / Cost and Usage Report は **24 時間ごとの更新**のため即時反映ではない

過去分のコストについて:

- 通常は、有効化後に Cost Explorer で見えるまで待ちます
- 過去分も必要な場合は、Billing の **Backfill tags** で **最大 12 か月**までバックフィルできます
- ただし、バックフィルで反映されるのは **その当時すでに対象リソースへタグが付いていた期間のコストだけ** です

## 9. Nova 2 Lite の `converse` 動作確認

Application Inference Profile ARN を `--model-id` に指定して実行します。

```bash
PROFILE_ARN=$(aws cloudformation describe-stacks \
  --stack-name application-inference-profile \
  --query "Stacks[0].Outputs[?OutputKey=='Nova2LiteInferenceProfileArn'].OutputValue | [0]" \
  --output text)

aws bedrock-runtime converse \
  --region ap-northeast-1 \
  --model-id "$PROFILE_ARN" \
  --messages '[{"role":"user","content":[{"text":"Please reply with exactly: Nova 2 Lite via application profile works."}]}]' \
  --inference-config '{"maxTokens":64,"temperature":0}' \
  --output json
```

`aws bedrock` と `jq` だけで ARN を取得して実行する場合:

```bash
PROFILE_ARN=$(aws bedrock list-inference-profiles \
  --region ap-northeast-1 \
  --type-equals APPLICATION \
  --output json \
| jq -r '
  .inferenceProfileSummaries[]
  | select(.inferenceProfileName == "application-inference-profile-apne1-nova-2-lite")
  | .inferenceProfileArn
')

aws bedrock-runtime converse \
  --region ap-northeast-1 \
  --model-id "$PROFILE_ARN" \
  --messages '[{"role":"user","content":[{"text":"Please reply with exactly: Nova 2 Lite via application profile works."}]}]' \
  --inference-config '{"maxTokens":64,"temperature":0}' \
  --output json
```

返り値例:

```json
{
  "output": {
    "message": {
      "role": "assistant",
      "content": [
        {
          "text": "Nova 2 Lite via application profile works."
        }
      ]
    }
  },
  "stopReason": "end_turn",
  "usage": {
    "inputTokens": 60,
    "outputTokens": 12,
    "totalTokens": 72
  },
  "metrics": {
    "latencyMs": 321
  }
}
```

## 10. 参考: 利用可能な Nova 系 System-Defined Inference Profile の確認

```bash
aws bedrock list-inference-profiles \
  --type-equals SYSTEM_DEFINED \
  --region ap-northeast-1 \
  --query "inferenceProfileSummaries[?contains(inferenceProfileName, 'Nova') || contains(inferenceProfileName, 'nova')].[inferenceProfileName,inferenceProfileArn,models[0].modelArn]" \
  --output table
```

この確認で `JP Amazon Nova 2 Lite` のコピー元 ARN を特定できます。

## 11. スタック削除

不要になったら CloudFormation スタックごと削除します。

```bash
aws cloudformation delete-stack \
  --stack-name application-inference-profile

aws cloudformation wait stack-delete-complete \
  --stack-name application-inference-profile
```

削除後の確認例:

```bash
aws cloudformation describe-stacks \
  --stack-name application-inference-profile
```

返り値例:

```text
aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id application-inference-profile does not exist
```

## 12. 補足

- CloudFormation の既存スタックは、テンプレートの Default 値を変えただけでは更新されません。
- 既存スタックのパラメータ値を変更したい場合は、`--parameter-overrides` を明示的に渡して再デプロイします。
- Application Inference Profile は `aws bedrock-runtime converse` の `--model-id` に ARN を直接指定して利用できます。
- `aws bedrock list-inference-profiles` の結果から `jq` で ARN や ID を取り出すと、CloudFormation Output に依存せず後続コマンドへ受け渡せます。
