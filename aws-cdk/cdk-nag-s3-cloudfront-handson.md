# cdk-nag ハンズオン手順記録

このプロジェクトは cdk-nag の学習用ハンズオン。作業内容を時系列で記録する。

## 環境

- CDK: `aws-cdk` ^2.1132.1 / `aws-cdk-lib` ^2.261.0
- TypeScript: 7.0.2(非常に新しいバージョンのため、`aws-cdk-lib` の型定義が要求する
  `Disposable` 型を解決するのに `lib` へ `esnext.disposable` の追加が必要だった)
- cdk-nag: ^3.0.1
- 実行方式: `tsx` で `app.ts` を直接実行(`ts-node` は未使用)

## 手順1: S3 + CloudFront (OAC) 静的サイト配信スタックの作成

`app.ts` に単一ファイルで `App` / `Stack` をまとめて定義。

- S3 バケット (`SiteBucket`)
  - `blockPublicAccess: BLOCK_ALL` でパブリックアクセスを禁止
  - `encryption: S3_MANAGED`
  - 学習用のため `removalPolicy: DESTROY` / `autoDeleteObjects: true`
- CloudFront Distribution (`Distribution`)
  - オリジンは `origins.S3BucketOrigin.withOriginAccessControl(siteBucket)` を使用し、
    OAC (Origin Access Control) 経由でオリジンアクセスを許可(レガシーな OAI は不使用)
  - `viewerProtocolPolicy: REDIRECT_TO_HTTPS`
  - SPA 向けに 404 を `index.html` にフォールバック
- `CfnOutput` でバケット名とディストリビューションのドメイン名を出力

この時点では **cdk-nag は未導入**。

合わせて以下の補助ファイルを作成:

- `tsconfig.json` — TypeScript 7 で `aws-cdk-lib` の型定義を解決するための設定
  (`lib: ["ES2022", "esnext.disposable"]` など)
- `cdk.json` — `"app": "npx tsx app.ts"` として CDK CLI のエントリポイントを指定

`npx cdk synth` で正常に synth できることを確認済み。

## 手順2: cdk-nag (AwsSolutions ルールパック) の追加

`app.ts` に以下を追加:

```typescript
import { AwsSolutionsChecks } from "cdk-nag";
...
cdk.Validations.of(app).addPlugins(new AwsSolutionsChecks(app, { verbose: true }));
```

### つまずいたポイント

手元の `cdk-nag@3.0.1` では、`NagPack`(`AwsSolutionsChecks` の基底クラス)が
`IAspect` ではなく CDK の新しい Validation Plugin フレームワーク
(`IPolicyValidationPlugin`)を実装する形に変わっていた。

- ❌ 旧来の書き方 `Aspects.of(stack).add(new AwsSolutionsChecks())` は
  `IAspect` の型不一致(`visit` メソッド不足)でコンパイルエラーになる。
- ✅ 代わりに `Validations.of(app).addPlugins(new AwsSolutionsChecks(app, { verbose: true }))`
  を使う必要がある。
  - プラグインは Stage または App スコープにのみ登録可能(Stack 単位では不可)。
  - `AwsSolutionsChecks` のコンストラクタは `(scope?, props?)` の順で、
    `scope` を渡すと `writeSuppressionsToCloudFormation` 使用時に
    `WriteNagSuppressionsToCloudFormationAspect` を自動で登録してくれる。

### 動作確認

`npx cdk synth` を実行し、以下の指摘が出ることを確認(この時点では未修正・未抑制):

| ルールID | 内容 | 重大度 |
|---|---|---|
| AwsSolutions-S1 | S3 バケットのアクセスログが無効 | ERROR |
| AwsSolutions-S10 | S3 バケット(ポリシー)が SSL 必須化していない | ERROR |
| AwsSolutions-CFR3 | CloudFront のアクセスログが無効 | ERROR |
| AwsSolutions-CFR4 | CloudFront がデフォルト証明書(TLSv1許容)を使用 | ERROR |
| AwsSolutions-CFR1 | Geo制限が未設定 | WARNING |
| AwsSolutions-CFR2 | WAF未連携 | WARNING |

`ERROR` が残っているため `cdk synth` はエラー終了する状態。

## 手順3: 各指摘事項の詳細調査

`cdk synth` で検出された6件について、それぞれ内容と対応方針を調査した。

### AwsSolutions-S1(ERROR): S3 バケットのアクセスログ無効

- **対象**: `StaticSiteStack/SiteBucket/Resource`
- **内容**: `SiteBucket` に `serverAccessLogsBucket` / `serverAccessLogsPrefix` が
  未設定のため、バケットへの各リクエスト(誰が・いつ・何を)の記録が残らない。
- **なぜ問題か**: インシデント発生時の追跡・監査証跡として、CIS/Well-Architected
  でも標準的に要求される項目。
- **対応方法**:
  - ① 専用のログ保存用バケット(`AccessLogsBucket`)を作成し
    `serverAccessLogsBucket` に指定する(推奨)。
    - ただしログ保存用バケット自身にも同ルールが再帰的に指摘されうるため、
      その場合はログバケット側は抑制するのが一般的。
  - ② 学習・検証用途でログ不要と判断するなら `NagSuppressions` で理由付き抑制。

### AwsSolutions-S10(ERROR): SSL(HTTPS)必須化されていない

- **対象**: `SiteBucket/Resource` と `SiteBucket/Policy/Resource` の2箇所
  (バケット本体とバケットポリシーそれぞれの観点でチェックされるため、
  原因は同一で指摘が2件出る)。
- **内容**: バケットポリシーに `aws:SecureTransport: false` の場合に `Deny` する
  条件が無く、理論上は平文HTTP接続も許容されてしまう。
- **なぜ問題か**: HTTP通信は暗号化されておらず、中間者攻撃(MITM)で盗聴・改ざん
  されるリスクがある。
- **対応方法**: CDK の `Bucket` に `enforceSSL: true` を1行追加するだけで、
  バケットポリシーに `SecureTransport` の `Deny` 条件が自動追加され、
  **2件同時に解消**する。副作用もほぼ無いため抑制ではなく修正を推奨。

### AwsSolutions-CFR3(ERROR): CloudFront アクセスログ無効

- **対象**: `StaticSiteStack/Distribution/Resource`
- **内容**: `Distribution` に `logBucket` が未設定のため、CDNエッジでの
  ビューアーリクエスト記録が残らない(オリジンであるS3側のログだけでは
  キャッシュヒット分が漏れる)。
- **対応方法**:
  - ① `enableLogging: true` + `logBucket`(S1対応のログバケット等を流用可)+
    `logFilePrefix` を指定する。バケット側の ACL/objectOwnership の組み合わせに
    よっては CloudFront からの書き込みが弾かれる場合があるため実機確認が必要。
  - ② 学習用途で不要なら `NagSuppressions` で抑制。

### AwsSolutions-CFR4(ERROR): TLSv1/SSLv3 を許容する設定

- **対象**: `StaticSiteStack/Distribution/Resource`
- **内容**: `minimumProtocolVersion` を明示していない(=デフォルトのCloudFront
  証明書を使用)場合、`SslSupportMethod` が `vip` 扱いとなり、
  **仕様上 TLSv1 が残ってしまう**(`minimumProtocolVersion` を指定しても効かない)。
- **重要な制約**: この問題を技術的に解消するには、**独自ドメイン + ACM証明書
  (us-east-1)** が必須。デフォルトドメイン(`*.cloudfront.net`)のままでは
  ルールを満たせない。
- **対応方法**:
  - ① 独自ドメイン・ACM証明書を用意し、`domainNames` / `certificate` /
    `minimumProtocolVersion: TLS_V1_2_2021` を設定する。
  - ② 独自ドメインを使わない学習用途では `NagSuppressions` で抑制するのが現実的。

### AwsSolutions-CFR1(WARNING): Geo制限未設定

- **対象**: `StaticSiteStack/Distribution/Resource`
- **内容**: `geoRestriction` が未設定。特定地域からのみアクセスを許可/拒否したい
  要件がある場合に検討すべき、という位置づけ(必須ではない)。
- **対応方法**:
  - ① 地域制限が必要なら `geoRestriction: cloudfront.GeoRestriction.allowlist(...)`
    を設定。
  - ② 全世界公開が要件なら `NagSuppressions` で抑制。

### AwsSolutions-CFR2(WARNING): WAF未連携

- **対象**: `StaticSiteStack/Distribution/Resource`
- **内容**: CloudFront に AWS WAF (WebACL) が関連付けられていない。
  アプリケーション層攻撃(SQLi/XSS/悪性IP等)への防御層として推奨される
  (必須ではない、コストも発生する)。
- **対応方法**:
  - ① WAF WebACL(`scope: "CLOUDFRONT"`、us-east-1で作成)を作成し
    `webAclId` に指定。
  - ② 小規模・学習用途では見送り、`NagSuppressions` で抑制。

## 手順4: ERROR を修正、WARNING を抑制

方針: **ERROR(S1, S10, CFR3, CFR4)は修正、WARNING(CFR1, CFR2)は抑制**という
指示だったが、CFR4 は仕様上「独自ドメイン+ACM証明書」が無いと技術的に解消
不可能(`cloudFrontDefaultCertificate === true` の場合は常に非準拠)なため、
ユーザーに確認したところ独自ドメインは無いとのことで、CFR4 も理由付きで
抑制する方針にした。

### 実施した修正

- **S10**: `siteBucket` に `enforceSSL: true` を追加(バケット本体・ポリシー
  両方の指摘が解消)。
- **S1**: アクセスログ格納用の `AccessLogsBucket` を新設し、`siteBucket` の
  `serverAccessLogsBucket` / `serverAccessLogsPrefix` に指定。
  - `AccessLogsBucket` にも `enforceSSL: true` を設定。
  - `AccessLogsBucket` 自身への S1 指摘(自己参照になるため)は抑制。
- **CFR3**: `Distribution` に `enableLogging: true` / `logBucket: accessLogsBucket`
  / `logFilePrefix` を追加。
  - CloudFront の標準ログ配信は ACL 書き込みを利用するため、`AccessLogsBucket`
    に `objectOwnership: s3.ObjectOwnership.OBJECT_WRITER` の指定が必要
    (`aws-cdk-lib` の `logBucket` プロパティのドキュメントコメントに明記あり)。

### 抑制した指摘(理由付き)

- **CFR4**(ERROR): 独自ドメイン・ACM証明書を使用しないため。
- **CFR1**(WARNING): 全世界公開想定のため地域制限不要。
- **CFR2**(WARNING): 学習用の小規模サイトのためWAF見送り。
- **S1 on AccessLogsBucket**: ログ格納先バケット自身へのログ設定は不要。

### つまずいたポイント: 抑制APIが `NagSuppressions` ではない

このバージョンの cdk-nag(3.0.1、CDK Validation Plugin フレームワーク対応版)
には従来の `NagSuppressions.addResourceSuppressions(...)` が**存在しない**
(`cdk-nag` の export 一覧に `NagSuppressions` が無い)。

代わりに CDK 本体の `Validations.of(scope).acknowledge(...)` を使う。

```typescript
cdk.Validations.of(construct).acknowledge({
  id: "AwsSolutions-S1",
  reason: "抑制理由",
});
```

- `acknowledge()` は可変長引数で複数ルールをまとめて渡せる。
- ID は **プレフィックス無しのルールID(例: `"AwsSolutions-S1"`)をそのまま渡す**。
  - synth 時のエラーメッセージには `Acknowledge with 'AwsSolutions::AwsSolutions-S1'`
    (`プラグイン名::ルールID` 形式)と表示されるが、これは表示上のヒントであり、
    実際に `acknowledge()` に渡すべきなのは `"AwsSolutions-S1"` という**バレ形式**
    だった(内部で自動的に `annotation::` プレフィックスが付与され、cdk-nag 側の
    照合ロジックでそのプレフィックスが除去されて突合される)。
    `"AwsSolutions::AwsSolutions-S1"` の形式でも動くかは未検証だが、バレ形式で
    `cdk synth` のERROR/WARNINGが実際に消えることを確認済み。
- `acknowledge()` は対象コンストラクト(またはその祖先)に対して呼ぶ。今回は
  `siteBucket` / `accessLogsBucket` / `distribution` それぞれの construct に対して実行。

### 動作確認

`npx cdk synth` を実行し、ERROR/WARNING が **1件も出力されないこと**、
また終了コードが `0`(成功)であることを確認済み。

## 現状

6件すべて対応済み。`cdk synth` はクリーンに成功する状態。
