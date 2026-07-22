# AWS CDK パターン集 ― cdk-patterns/serverless と aws-samples

作成日: 2026-07-19

対象: [cdk-patterns/serverless](https://github.com/cdk-patterns/serverless)、[aws-samples](https://github.com/aws-samples)、[Serverless Land Patterns](https://serverlessland.com/patterns)、CLIツール `cdkp`

## 学んだこと（元メモ）

- サーバーレスのCDKパターン集が [cdk-patterns/serverless](https://github.com/cdk-patterns/serverless) にある。
- AWS公式のサンプルコード集として [aws-samples](https://github.com/aws-samples) がある。
- `npx cdkp list` でパターンの一覧を確認できる。
- パターン集として [Serverless Land Patterns](https://serverlessland.com/patterns) というサイトもある。

## cdk-patterns/serverless とは

AWSのサーバーレスアーキテクチャパターンを、実際にデプロイ可能なCDKコードとして集めたOSSリポジトリ。個人（Matt Coulter氏）が中心となって運営しているコミュニティ製リソースで、AWS公式リポジトリではない点に注意。

- 各パターンは **TypeScript** と **Python** の両方で提供され、エクスポートされた **CloudFormationテンプレート** も同梱される。
- 掲載サイトは [cdkpatterns.com](https://cdkpatterns.com/) で、Well-Architectedフレームワークの柱（信頼性・コスト最適化など）に沿った分類で紹介されている。
- 執筆時点で30種類以上のパターンが公開されている。

### 代表的なパターン例

| パターン名 | 概要 |
|---|---|
| The Simple Webservice | API Gateway + Lambda + DynamoDBの基本形 |
| The Lambda Trilogy | Lambda実装パターンの3バリエーション比較 |
| EventBridge ATM | 条件に応じたイベントルーティング |
| The State Machine | Step Functionsによる複雑なオーケストレーション |
| Polly | Lambda内でテキスト読み上げ（Amazon Polly連携） |
| RDS Proxy | Lambda→RDSアクセスをProxyで保護 |
| S3 Angular/React Deploy | フロントエンドの高速デプロイ構成 |

## `cdkp` CLIコマンド

`cdkp` は [npmパッケージ](https://www.npmjs.com/package/cdkp) として配布されており、リポジトリを丸ごとcloneせず、必要なパターンだけを取得できるのが利点。

```bash
# 利用可能なパターンを一覧表示
npx cdkp list

# 指定パターンを取得（デフォルトはTypeScript）
npx cdkp init {pattern-name}

# Python版を取得する場合
npx cdkp init {pattern-name} --lang=python
```

取得後は通常のCDKプロジェクトと同様に、`cd {pattern-name}` してから `npm run test` や `npm run deploy`（あるいは `cdk deploy`）で動作確認できる。

## aws-samples とは

AWSが公式に運営するGitHub Organizationで、8,000を超えるリポジトリを抱える大規模なサンプルコード集。「サポート対象の製品ではなく教育目的の例示」であり、実運用時は自社のセキュリティ基準に沿ったテストが必須とされている点が明記されている。

CDK関連では **[aws-samples/aws-cdk-examples](https://github.com/aws-samples/aws-cdk-examples)** がピン留めされた主要リポジトリで、様々なユースケースのCDKプロジェクト例を多言語（TypeScript/Python/Java/C#等）で提供している。

## Serverless Land Patterns とは

[serverlessland.com/patterns](https://serverlessland.com/patterns) はAWS公式が運営するサーバーレス学習ポータル「Serverless Land」内のパターン集ページ。バックエンドの実体は同じくAWS公式Organization配下の [aws-samples/serverless-patterns](https://github.com/aws-samples/serverless-patterns) リポジトリで、サイトはそのカタログのフロントエンドという位置づけ。

- 執筆時点で **約240パターン**、対象サービスは30種類以上と、他の2つより収録数が多い。
- **CDKに限定されない** のが最大の特徴で、AWS SAM／CDK／Terraform／Serverless Frameworkなど複数のIaCフレームワークに対応し、サイト上でフレームワークやサービスを指定して絞り込める。
- 2025年3月には **VS Code拡張との連携** が追加され、IDE上でパターンを検索・フィルタし、そのままワークスペースにダウンロードできるようになった。

## 3サイトの使い分け

| | cdk-patterns/serverless | aws-samples/aws-cdk-examples | Serverless Land Patterns |
|---|---|---|---|
| 運営元 | コミュニティ（個人OSS） | AWS公式 | AWS公式 |
| 対象領域 | サーバーレスパターンに特化 | CDK全般（サーバーレス以外も含む） | サーバーレス統合パターン全般（IaC横断） |
| 対応IaC | CDK（TS/Python） | CDK（多言語） | SAM／CDK／Terraform／Serverless Framework |
| 取得方法 | `cdkp` CLIで個別パターンをダウンロード | リポジトリをcloneしてサブディレクトリを参照 | サイト/VS Code拡張からパターン単位で取得 |
| 学習に向く場面 | 「このユースケースの定番構成を知りたい」時 | 「特定サービス・言語のCDK実装例を探したい」時 | 「IaCツールを問わず統合パターンを広く探したい」時 |

## 次のハンズオン: The CloudWatch Dashboard

「運用監視」をテーマにハンズオンで学ぶパターンとして、**cdk-patterns/serverless の [The CloudWatch Dashboard](https://github.com/cdk-patterns/serverless/blob/main/the-cloudwatch-dashboard/README.md)** を選定。

- ベースは The Simple Webservice（API Gateway + Lambda + DynamoDB）で、そこに本番運用を意識した監視基盤を追加する構成。
- 追加される要素: **CloudWatchダッシュボード**、**アラーム**（例: Lambda p99レイテンシ ≥ 1秒）、通知用の **SNSトピック**。
- 学べるポイント:
  - メトリクスの4分類（ビジネス／カスタマーエクスペリエンス／システム／運用）という監視設計の考え方
  - Metric Math（複数メトリクスの数式演算）の使い方
  - ディメンション・統計値（p99など）の適切な選び方
  - アラームとSNS通知を組み合わせたアラーティングの実装

### 始め方

```bash
npx cdkp init the-cloudwatch-dashboard
# Python版の場合
npx cdkp init the-cloudwatch-dashboard --lang=python

cd the-cloudwatch-dashboard
npm install
npm run test
cdk deploy   # AWSアカウントへの実デプロイ（課金・実リソース作成が発生する点に注意）
```

### 次の候補（運用監視の発展編）

CloudWatch Dashboardで基礎を押さえた後は、Serverless Landの以下のパターンで発展的な内容に進むのも良い。

- [CloudWatch Logs → SNS/SQS通知](https://github.com/aws-samples/serverless-patterns/tree/main/cwlogs-lambda-sns-sqs-cdk)（サブスクリプションフィルタでエラーログを検知してアラート）
- [EventBridge → Lambda + CloudWatch Alarmフィードバック](https://serverlessland.com/patterns/cdk-closed-loop-serverless-control-pattern)（アラームの状態変化を起点に自動制御するClosed Loopパターン）

## 参考リンク

- [cdk-patterns/serverless (GitHub)](https://github.com/cdk-patterns/serverless)
- [AWS CDK Patterns 公式サイト](https://cdkpatterns.com/)
- [cdkp (npm)](https://www.npmjs.com/package/cdkp)
- [aws-samples (GitHub Organization)](https://github.com/aws-samples)
- [aws-samples/aws-cdk-examples (GitHub)](https://github.com/aws-samples/aws-cdk-examples)
- [Serverless Land Patterns](https://serverlessland.com/patterns)
- [aws-samples/serverless-patterns (GitHub)](https://github.com/aws-samples/serverless-patterns)
- [Serverless Land PatternsのVS Code連携 (AWS What's New)](https://aws.amazon.com/about-aws/whats-new/2025/03/ready-to-use-serverless-land-patterns-vs-code-ide/)
