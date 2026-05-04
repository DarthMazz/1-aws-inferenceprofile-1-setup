# 1-aws-inferenceprofile-1-setup

AWS Inference Profile の作成、および AWS CLI / GitHub CLI (`gh`) を使った確認作業を行うためのリポジトリです。

## 目的

- AWS Inference Profile の作成手順を整理する
- AWS CLI で必要な確認コマンドを実行する
- GitHub CLI (`gh`) でリポジトリや認証状態を確認する

## 前提

- `aws` コマンドが利用できること
- `gh` コマンドが利用できること
- AWS 認証情報が設定済みであること
- `gh` を利用する場合は `gh auth login` もしくは `GH_TOKEN` の設定が済んでいること

## 動作確認コマンド

### AWS CLI

```bash
aws --version
aws sts get-caller-identity
```

### GitHub CLI

```bash
gh --version
gh auth status
```

## 作業メモ

- Inference Profile の作成手順や確認コマンドは、このリポジトリに今後追記していきます。
- 実際の AWS リソース作成前に、対象リージョンと利用するモデルを明確にしておく想定です。
