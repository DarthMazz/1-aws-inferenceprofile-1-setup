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

## 5. 作成したプロファイルの詳細確認

### Claude Haiku 4.5

```bash
PROFILE_ARN=$(aws cloudformation describe-stacks \
  --stack-name application-inference-profile \
  --query "Stacks[0].Outputs[?OutputKey=='InferenceProfileArn'].OutputValue | [0]" \
  --output text)

aws bedrock get-inference-profile \
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

## 6. Nova 2 Lite の `converse` 動作確認

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

## 7. 参考: 利用可能な Nova 系 System-Defined Inference Profile の確認

```bash
aws bedrock list-inference-profiles \
  --type-equals SYSTEM_DEFINED \
  --region ap-northeast-1 \
  --query "inferenceProfileSummaries[?contains(inferenceProfileName, 'Nova') || contains(inferenceProfileName, 'nova')].[inferenceProfileName,inferenceProfileArn,models[0].modelArn]" \
  --output table
```

この確認で `JP Amazon Nova 2 Lite` のコピー元 ARN を特定できます。

## 8. スタック削除

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

## 補足

- CloudFormation の既存スタックは、テンプレートの Default 値を変えただけでは更新されません。
- 既存スタックのパラメータ値を変更したい場合は、`--parameter-overrides` を明示的に渡して再デプロイします。
- Application Inference Profile は `aws bedrock-runtime converse` の `--model-id` に ARN を直接指定して利用できます。
