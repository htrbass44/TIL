# Terraform ハンズオン — 「動く」から「配れる」へ、社内クラウドサービスカタログを GitLab に集める

> **対象読者**: Terraform を触ったことはあるが、`terraform apply` が通って以降の「チームで使う形」に自信がない人。あるいは一度学んで忘れた人。
> **最終ゴール**: 社内の開発チームが **GitLab に MR を1本出すだけで AWS リソースが払い出される** 状態を、自分の手で一度作りきる。
> **所要時間**: 約5時間（Part A 3時間 + Part B 2時間、分割実施を推奨）
> **ターミナル**: GitBash（Windows）
> **クラウド**: AWS（`ap-northeast-1`）
> **想定費用**: 全部やっても **月額数十円未満**（EC2・NAT Gateway を一切使わない構成）

---

## 目次

1. [勉強対象の概要](#1-勉強対象の概要)
2. [ハンズオンの概要](#2-ハンズオンの概要)
3. [ハンズオンの手順](#3-ハンズオンの手順)
4. [習得事項のまとめ](#4-習得事項のまとめ)
5. [今後の学習ロードマップ](#5-今後の学習ロードマップ)

---

## 1. 勉強対象の概要

### 1.1 Terraform を「AWS CLI の便利版」だと思っていると必ず詰まる

Terraform を「設定ファイルを書くと AWS にリソースが作られるツール」と説明されることが多い。それは間違ってはいないが、**その理解のままだとチーム運用に入った瞬間に事故る**。

正確には次のとおり。

> Terraform は、**3つの世界の差分を計算して埋めるだけのツール**である。
> ① `.tf` に書いた「あるべき姿」、② state に記録された「自分が管理しているもの」、③ クラウド上の「現実」。
> **`plan` は ①と② の差**を出し、**`apply` はその差を ③ に反映して ② を書き直す**。

![Terraform の中心概念](diagrams/01-concept-map.drawio.png)

この図で最も重要なのは、**赤枠の ② State** である。Terraform が「自分が何を管理しているか」を知る唯一の手段が state であり、

- **state に無いものは、AWS に実在していても Terraform には見えない**（だから `destroy` されない）
- **state にあるのに AWS に無いものは、次の `plan` で「作り直す」と言われる**
- **state が壊れる／消える＝管理権を失う**（リソースは残るが Terraform から触れなくなる）

「Terraform 難しい」と言われるトラブルの体感 8 割は、この state をめぐる問題である。だから本ハンズオンは、**演習1でクラウドを一切使わずに state だけを観察する**ところから始める。

### 1.2 中心概念は7つだけ

| 概念 | 一言でいうと | 書く場所 | 本ハンズオンで扱う演習 |
|------|-------------|---------|----------------------|
| **Provider** | AWS などの API を叩くプラグイン | `required_providers` / `provider` ブロック | 演習1, 2 |
| **Resource** | 作りたいモノ1つ | `resource` ブロック | 演習1, 2 |
| **Data Source** | 既にあるモノを読むだけ | `data` ブロック | 演習5 |
| **State** | いま管理しているものの台帳 | `terraform.tfstate`（自動生成） | **演習1**, 4 |
| **Backend** | state の置き場所 | `terraform { backend "s3" {} }` | 演習4 |
| **Variable / Output** | 入力パラメータと出力値 | `variable` / `output` ブロック | 演習3 |
| **Module** | 再利用できる部品の単位 | ディレクトリ + `module` ブロック | **演習5〜9** |

この7つ以外（`locals`, `for_each`, `dynamic`, `lifecycle`, `moved` …）はすべて、**この7つを楽に書くための道具**にすぎない。順番を間違えて `for_each` から覚えると混乱する。

### 1.3 HCL のブロックは6種類しかない

`.tf` ファイルに書けるトップレベルのブロックは、実質これだけである。

```hcl
terraform { ... }   # Terraform 自身の設定（バージョン制約・backend）
provider  { ... }   # プロバイダの設定（リージョン・認証・共通タグ）
variable  { ... }   # 入力（＝この構成の「引数」）
locals    { ... }   # 計算した中間値（変数ではなく「式の名前付け」）
resource  { ... }   # 作るもの
data      { ... }   # 読むだけのもの
output    { ... }   # 出力（＝この構成の「戻り値」）
module    { ... }   # 他のディレクトリを呼び出す
```

> **腹落ちのコツ**: 1つのディレクトリを **「引数(`variable`)を取り、副作用(`resource`)を起こし、戻り値(`output`)を返す関数」** だと思うと、モジュール化の話が一気に分かりやすくなる。カタログの「商品」とは、この関数のことである。

ファイル名（`main.tf` / `variables.tf` / `outputs.tf`）に文法上の意味はない。**同一ディレクトリの `.tf` は全部まとめて1つとして読まれる**。分けるのは人間のためだけの慣習である。

### 1.4 実行のライフサイクルと、plan の記号の読み方

![実行のライフサイクル](diagrams/02-plan-apply-cycle.drawio.png)

`plan` の出力で**最初に探すべきは `-/+`（replace）と `-`（destroy）の2つ**。この2つが出ていないかだけ見れば、事故の大半は防げる。

逆に最も危険な癖が、末尾のサマリ行だけを読むことである。

```
Plan: 2 to add, 0 to change, 1 to destroy.
```

この「1 to destroy」が、テスト用のダミーなのか本番の RDS なのかは、**サマリ行からは絶対に分からない**。演習2でこの読み方を身体に入れる。

### 1.5 「モジュール」はカタログの商品である

ここが本ハンズオンの背骨になる考え方。

| カタログの言葉 | Terraform の言葉 | 実体 |
|---------------|-----------------|------|
| 商品（例: 「標準ログバケット」） | **Module** | `modules/s3-bucket/` ディレクトリ |
| 商品カタログの棚 | **Module Registry** | GitLab の Terraform Module Registry |
| 商品の型番 | **version** | Git タグ `0.1.0` |
| 注文フォーム | **variable + validation** | `variable "environment" { validation {...} }` |
| 「これは注文できません」 | **validation エラー** | `plan` の時点で落ちる |
| レジ／出荷 | **CI/CD の apply ジョブ** | `.gitlab-ci.yml` |
| 納品書 | **output** | バケット名・ARN |

つまり **「社内クラウドサービスカタログを作る」＝「良いモジュールを書いて、GitLab にバージョン付きで並べ、CI から呼べるようにする」** に分解できる。難しいのは Terraform ではなく、**どこまでを利用者に選ばせ、どこを固定するかという設計**のほうである（4.3 で扱う）。

### 1.6 セルフサービスカタログの3類型

社内カタログと言ったとき、実際には3つの別物が混ざっている。今回作るのは **B** だが、A・C との違いを知らずに始めると設計を間違える。

| 型 | 利用者がやること | 向いている場面 | Terraform 上の実装 |
|----|-----------------|--------------|-------------------|
| **A. モジュール配布型** | 自分のリポジトリで `module` を呼び、自分で apply する | 各チームが自分の AWS アカウントと state を持っている | Module Registry に publish するだけ |
| **B. セルフサービス払い出し型**（今回） | 申請ファイルを1枚追加した MR を出す。apply はプラットフォーム側の CI | 中央のチームが払い出しを管理したい | Registry + カタログリポジトリ + CI |
| **C. 棚卸し・可視化型** | 何もしない（既存資源を後追いでコード化） | まず現状を把握したい | `import` ブロック / `terraform query` |

> **ここが実務上の分かれ目**: A は「配って終わり」なので楽だが、**各チームに Terraform の学習コストと apply 権限を配る**ことになる。B は中央に運用負荷が集まる代わりに、**利用者は Terraform を一切知らなくてよく、権限も中央に閉じ込められる**。「クラウドサービスカタログ」という言葉で期待されているのはたいてい B である。

### 1.7 2026年時点の分岐 — Terraform か OpenTofu か

学び直しの前に、一度だけ確認しておくべき事実がある。

| | Terraform | OpenTofu |
|---|-----------|----------|
| 開発元 | HashiCorp（IBM） | Linux Foundation |
| ライセンス | **BUSL 1.1**（商用の競合利用に制限） | **MPL 2.0**（従来の Terraform と同じ） |
| CLI | `terraform` | `tofu` |
| HCL 文法 | ほぼ共通（1.5系までは互換） | ほぼ共通 |
| GitLab の公式 CI/CD 部品 | **提供終了**（Terraform CI/CD テンプレートと `terraform-images` の配布は終了） | **[OpenTofu CI/CD コンポーネント](https://gitlab.com/components/opentofu)** が現行の推奨 |
| GitLab Module Registry | **両方から利用可**（Module Registry Protocol は共通） | 同左 |

つまり、**「Terraform を学ぶ」ことと「GitLab の公式部品に乗る」ことは、2026年時点では少しズレている**。

本ハンズオンの方針は次のとおり。

- **文法・概念・モジュール設計はすべて共通**なので、Terraform で学ぶ。ここで学んだことは OpenTofu にそのまま通用する。
- **CI は自前ジョブ（`hashicorp/terraform` イメージ）で書く方法を主として示す**。GitLab の公式テンプレートが無くても、20行程度で書けることを確認する。
- 演習8の最後に、**OpenTofu コンポーネントに乗り換える場合の差分**を併記する。社内標準を決める際の材料にしてほしい。
- なお **モジュールの publish 用テンプレート（`Terraform-Module.gitlab-ci.yml`）は現在も提供されている**。廃止されたのは「デプロイ用」のテンプレートである。ここは混同しやすいので注意。

---

## 2. ハンズオンの概要

### 2.1 ゴールイメージ

最終的に、次の状態を自分の GitLab / AWS 上に作る。

![完成形のアーキテクチャ](diagrams/04-catalog-architecture.drawio.png)

利用者（開発チーム）の体験は、最終的にこれだけになる。

```jsonc
// requests/team-alpha/logs.json を追加する MR を出す。それだけ。
{
  "name":        "logs",
  "environment": "dev",
  "owner_team":  "team-alpha",
  "cost_center": "CC-1234",
  "retention_days": 30
}
```

| 状態 | Before（よくある状態） | After（このハンズオンの完成形） |
|------|---------------------|---------------------------|
| リソースの作り方 | マネジメントコンソールで手作業 | MR 1本 |
| 命名規則 | Wiki に書いてあるが守られない | `validation` で機械的に強制 |
| 暗号化・公開ブロック | 「チェック忘れ」が起きる | モジュール側で固定、利用者は選べない |
| 誰のリソースか | 分からない | 必須タグが無いと `plan` が落ちる |
| AWS の認証情報 | CI 変数にアクセスキーを保存 | **OIDC。長期キーはゼロ** |
| 変更履歴 | CloudTrail を掘る | Git の履歴と MR のレビューコメント |

### 2.2 学べることの全体像

| レイヤ | 学ぶこと | 該当演習 |
|--------|---------|---------|
| **Terraform の芯** | state とは何か、plan の読み方、ドリフト | 演習1, 2 |
| **設定の作法** | 変数・型・validation・output・優先順位 | 演習3 |
| **チーム開発** | リモート state、ロック、backend 移行 | 演習4 |
| **部品化** | モジュール設計、provider を書かない理由、ガードレール | 演習5 |
| **配布** | GitLab Module Registry、semver、認証 | 演習6, 7 |
| **自動化** | MR で plan / merge で apply、OIDC、artifact の秘匿 | 演習8 |
| **カタログ運用** | 申請ファイル駆動、`for_each`、CODEOWNERS、権限分離 | 演習9 |

### 2.3 演習の流れ

![演習の流れ](diagrams/03-exercise-flow.drawio.png)

**Part A（演習0〜5）は AWS の課金がほぼ発生しない**ので、まとまった時間が取れなくても進めやすい。**Part B（演習6〜10）は GitLab プロジェクトを3つ使う**ので、先に作っておくと流れが途切れない。

### 2.4 費用について

| 使うもの | 課金 | 概算 |
|---------|------|------|
| S3 バケット（空 or 数KB） | ストレージ + リクエスト | **1円/月未満** |
| S3 バックエンド（state 保管 + ロックファイル） | 同上 | **1円/月未満** |
| GitLab Terraform Module Registry | パッケージレジストリのストレージ | Free プランの枠内 |
| GitLab CI 実行時間 | 共有ランナーの分数 | Free プランの月次枠内（本演習で数分） |
| IAM / OIDC プロバイダ | 無料 | 0円 |

> **注意**: 費用がほぼゼロなのは **EC2・NAT Gateway・RDS を一切使わない**設計にしているため。演習を自分の題材に置き換えるとき、NAT Gateway（約 $0.062/時 ≒ 月7,000円弱）と Elastic IP の未使用課金には特に注意すること。**演習10のクリーンアップは必ず実施する。**

### 2.5 前提環境

| 必要なもの | 確認コマンド | 備考 |
|-----------|------------|------|
| Terraform 1.10 以上 | `terraform version` | 本文は v1.14 で確認。`use_lockfile` に 1.10 以上が必要 |
| AWS CLI v2 | `aws --version` | 認証確認と後片付けに使う |
| AWS アカウント | `aws sts get-caller-identity` | 検証用アカウント推奨 |
| GitLab アカウント | — | GitLab.com（Free）で可。**トップレベルグループを1つ作れること** |
| Git | `git --version` | — |
| jq（あると便利） | `jq --version` | state を覗くのに使う |

**作業ディレクトリ**: 本ハンズオンは `/c/dev/handson-terraform-beginner/tf-handson`（＝ `C:\dev\handson-terraform-beginner\tf-handson`）配下で作業する。以降のコマンドはすべてこのパスを前提にしている。

---

## 3. ハンズオンの手順

### 演習0: 事前準備

**目的**: 手を動かす前に、詰まる要因を潰しておく。

#### 0-1. バージョンと認証を確認する

```bash
terraform version
aws --version
aws sts get-caller-identity
```

`terraform version` で次のような「最新版ではない」旨のメッセージが出ることがある。

```
Terraform v1.14.1
on windows_amd64

Your version of Terraform is out of date! The latest version
is 1.16.1. You can update by downloading from https://developer.hashicorp.com/terraform/install
```

**このまま進めてよい。** 本ハンズオンが要求するのは **1.10 以上**（`use_lockfile` と、変数間の `validation` に必要）であり、`terraform version` はバージョンチェックのたびに最新版の存在を知らせてくるだけである。エラーではない。

> **むしろ、安易に上げないほうがよい理由がある**。**Terraform の state には、書き込んだバージョンが記録される。新しいバージョンで書いた state は、古いバージョンから読めない。**
>
> ```
> Error: state snapshot was created by Terraform v1.16.1, which is newer than
> current v1.14.1; upgrade to Terraform v1.16.1 or greater to work with this state
> ```
>
> つまり **state は「一度上げたら下げられない」**。個人の学習では気にならないが、**チームで1つの state を共有した瞬間に、これは全員を巻き込む事故になる**。誰か1人がローカルで新しい Terraform を使って apply すると、他のメンバーと CI が全員止まる。
>
> **カタログ運用における対策は2つ。**
> 1. **CI の Terraform バージョンを唯一の正とする**（本教材の `.gitlab-ci.yml` は `hashicorp/terraform:1.14` を指定している）。ローカルはそれと同じかそれ以下に揃える。
> 2. **`required_version` で下限だけでなく上限も切る**。カタログ本体では次のように書いておくと、想定外のバージョンでの apply を防げる。
>    ```hcl
>    terraform {
>      required_version = "~> 1.14"   # 1.14.x のみ許可。1.15 以降では実行できない
>    }
>    ```
>
> バージョンを上げるときは、**CI のイメージタグ → `required_version` → 各自のローカル、の順で足並みを揃えて上げる**。これが「学び直し」で最も見落とされがちな運用作法である。

`get-caller-identity` が次のように返れば OK。`Account` の値は後で何度も使うのでメモしておく。

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-name"
}
```

#### 0-2. 作業ディレクトリを作る

本ハンズオンの作業ディレクトリは **`/c/dev/handson-terraform-beginner/tf-handson`**（Windows 表記で `C:\dev\handson-terraform-beginner\tf-handson`）で統一する。以降のコマンドはすべてこのパスを前提に書いてある。

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson
cd /c/dev/handson-terraform-beginner/tf-handson
pwd    # /c/dev/handson-terraform-beginner/tf-handson と出れば OK
```

演習ごとに、この下にサブディレクトリを作っていく。

```
/c/dev/handson-terraform-beginner/tf-handson/
├── ex1/                    ← 演習1（state の観察）
├── ex2/                    ← 演習2（AWS に1つ作る）
├── ex3/                    ← 演習3・4（変数、リモート state）
├── ex5/                    ← 演習5（モジュールの動作確認）
├── ex7/                    ← 演習7（Registry 経由の呼び出し）
├── tf-module-s3-bucket/    ← 演習5・6（モジュール本体。GitLab リポジトリになる）
└── tf-catalog/             ← 演習8・9（カタログ本体。GitLab リポジトリになる）
```

> **GitBash のパス表記について**: GitBash では `C:\` を `/c/` と書く。PowerShell や cmd を併用する場合は `C:\dev\handson-terraform-beginner\tf-handson` に読み替えること。`cd /c/dev/...` が「No such file or directory」になる場合は、ドライブレターの大文字小文字（`/C/` ではなく `/c/`）を確認する。

#### 0-3. Windows 固有の設定を2つ入れておく

**(a) 改行コード**: リポジトリ内を LF に揃えておく。各リポジトリ（`tf-module-s3-bucket` / `tf-catalog`）の**直下**に `.gitattributes` を置く。

> **`terraform fmt` は改行コードに触らない。** CRLF のままでも `terraform fmt -check` は合格する。揃える理由は別にあって、
> - Windows と Linux の参加者が混在したとき、**改行だけの差分でレビューが埋まる**のを防ぐ
> - シェルスクリプトを追加したとき、CRLF のまま Linux コンテナで実行されて `#!/bin/sh^M: not found` になるのを防ぐ
>
> なお push 時に出る次の警告は**正常**である。「コミットには LF で入った。次のチェックアウトで作業コピーは CRLF になる」という予告にすぎない。リポジトリ側は LF で正規化されている。
> ```
> warning: in the working copy of 'main.tf', LF will be replaced by CRLF the next time Git touches it
> ```

```bash
cat > .gitattributes <<'EOF'
* text=auto eol=lf
*.tf   text eol=lf
*.tfvars text eol=lf
EOF
```

**(b) エディタの自動整形**: VS Code なら HashiCorp Terraform 拡張を入れておくと `terraform fmt` 相当が保存時に走る。

> **確認ポイント**: `terraform version` が 1.10 未満なら演習4の `use_lockfile` が使えない。[公式サイト](https://developer.hashicorp.com/terraform/install)から更新するか、演習4だけ DynamoDB 方式（後述の補足）で進める。

---

### 演習1: state を裸で観察する（AWS を使わない）

**目的**: **Terraform の挙動が state で決まる**ことを、課金もリスクもない環境で体に入れる。ここを飛ばすと、後半のトラブルが全部「よく分からない現象」になる。

#### 1-1. 最小の構成を書く

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/ex1 && cd /c/dev/handson-terraform-beginner/tf-handson/ex1
```

`main.tf`:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

resource "random_pet" "name" {
  length    = 2
  separator = "-"
}

resource "local_file" "greeting" {
  filename        = "${path.module}/hello.txt"
  content         = "hello, ${random_pet.name.id}\n"
  file_permission = "0644"
}
```

#### 1-2. init して plan を読む

```bash
terraform init
terraform plan
```

出力に注目する。

```
Terraform will perform the following actions:

  # local_file.greeting will be created
  + resource "local_file" "greeting" {
      + content              = (known after apply)
      + filename             = "./hello.txt"
      + id                   = (known after apply)
      ...
    }

  # random_pet.name will be created
  + resource "random_pet" "name" {
      + id        = (known after apply)
      + length    = 2
      + separator = "-"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

`content = (known after apply)` になっているのは、**`random_pet` の結果が出るまで確定しないから**。Terraform は `${random_pet.name.id}` という参照から**依存関係のグラフを自動で組み立てている**。`depends_on` を書く必要は基本的にない。

#### 1-3. apply して state を覗く

```bash
terraform apply -auto-approve
cat hello.txt
```

ここからが本題。生の state を見る。

```bash
terraform state list
```

```
local_file.greeting
random_pet.name
```

**この2行が「Terraform が管理していると思っているもの」の全て**である。1つずつ中身を見る。

```bash
terraform state show local_file.greeting
```

```
# local_file.greeting:
resource "local_file" "greeting" {
    content              = <<-EOT
        hello, sacred-mullet
    EOT
    content_md5          = "1beb4c6a7a45d3bf5a556d6ae45563bb"
    content_sha1         = "9325bd5d052184c615fbfea3ab955275cecea4b6"
    content_sha256       = "d5a6ce9891b8c76302e953d05c8c0c939c0711027d22e45330531e7d2fb28d34"
    directory_permission = "0777"
    file_permission      = "0644"
    filename             = "./hello.txt"
    id                   = "9325bd5d052184c615fbfea3ab955275cecea4b6"
}
```

ファイルそのものも見ておく（**普段は直接編集しないこと**）。

```bash
cat terraform.tfstate | jq '.resources[] | {type, name, id: .instances[0].attributes.id}'
```

```json
{ "type": "local_file", "name": "greeting", "id": "9325bd5d052184c615fbfea3ab955275cecea4b6" }
{ "type": "random_pet", "name": "name",     "id": "sacred-mullet" }
```

> **ここで必ず引っかかる点 — `id` はファイル名ではない**
>
> `local_file.greeting` の `id` が 40 桁の16進数になっているのを見て、「これが `hello.txt` に対応しているのか？」と迷うのは正しい反応である。答えは **Yes、この1行が `hello.txt` の台帳エントリ**。ただし **`id` はファイルのパスではなく、`local` プロバイダが決めた「中身の SHA1 ハッシュ」**である。パスは別の属性 `filename` に入っている。
>
> 自分で確かめられる。
>
> ```bash
> sha1sum hello.txt
> # 9325bd5d052184c615fbfea3ab955275cecea4b6 *hello.txt   ← state の id と一致する
> ```
>
> **`id` の中身は、リソースの種類ごとに完全にバラバラ**である。「Terraform 共通の ID 体系」というものは存在しない。
>
> | リソース | `id` の実体 |
> |---------|-----------|
> | `local_file` | **中身の SHA1 ハッシュ** |
> | `random_pet` | 生成された文字列そのもの（`sacred-mullet`） |
> | `aws_s3_bucket` | バケット名（`acme-dev-logs-123456789012`） |
> | `aws_instance` | インスタンス ID（`i-0a1b2c3d4e5f`） |
> | `aws_iam_role` | ロール名 |
>
> `id` は **「そのプロバイダが、そのリソースを再び見つけるために使う識別子」** でしかない。だから **`id` を見て何かを判断してはいけない**。人間が見るべきは `filename` や `bucket` のような意味のある属性のほうである。
>
> なお、演習1-4 で `hello.txt` を書き換えると `id` も変わる（中身が変われば SHA1 が変わるため）。これは `local_file` に固有の挙動で、S3 バケットの `id` は中身を変えても不変である。

> **腹落ちポイント**: `terraform state list` の2行が ②（管理台帳）、`.tf` が ①（あるべき姿）、`hello.txt` という実ファイルが ③（現実）。この3つの区別が、この先ずっと効いてくる。

#### 1-4. ドリフト（現実がズレた状態）を起こす

```bash
rm hello.txt
terraform plan
```

```
random_pet.name: Refreshing state... [id=fresh-gnu]
local_file.greeting: Refreshing state... [id=a383b7aeb0c32b0188b191af2e1bfe0140d647ad]

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.greeting will be created
  + resource "local_file" "greeting" {
      + content              = <<-EOT
            hello, fresh-gnu
        EOT
      + content_sha1         = (known after apply)
      + file_permission      = "0644"
      + filename             = "./hello.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

見るべきところは3つ。

| 出力 | 意味 |
|------|------|
| `Refreshing state... [id=...]` | **plan の最初に必ず現実（③）を読み直している**。これが `refresh` |
| `# local_file.greeting will be created` | 読み直した結果「無い」と分かったので、作る計画になった |
| `content = hello, fresh-gnu` | **ペット名が変わっていない**。値は state に残っているので、消えたファイルだけが復元される（詳細は 1-6） |

> **「ドリフト検知」なのに `Note: Objects have changed outside of Terraform` が出ないのはなぜか**
>
> あの `Note:` ブロックは、**オブジェクトがまだ存在していて、属性の値だけが変わった**ときに出る。今回はファイルごと消えており、リフレッシュの結果 `local_file.greeting` は **state から取り除かれた**。state に無いものは単に「作る」対象であって、「変更された」ものではない。だから通常の `+ create` として表示される。
>
> 消したのではなく**中身を書き換えた**場合も、`local_file` に限っては同じく `+ create` になる。1-3 で見たとおり **`local_file` の `id` は中身の SHA1** なので、中身が変わると「別のオブジェクト＝元のものは消えた」とプロバイダが判断するためである。
>
> ```bash
> echo "tampered" > hello.txt
> terraform plan     # やはり + create になる
> ```
>
> `Note:` ブロックのほうは **演習2-5** で見る。S3 のバケットは存在したまま設定値だけが変わるので、そちらが本来の「変更された」ケースになる。

いずれにせよ本質は同じ。**Terraform は plan の最初に必ず現実（③）を読み直し、state（②）を実態に合わせてから、設定（①）との差を出す。**

読み直しをスキップするとどうなるかも見ておく。

```bash
terraform plan -refresh=false
```

```
No changes. Your infrastructure matches the configuration.
```

**現実を見ずに ① と ② だけ比べると「変更なし」になる。** CI で `-refresh=false` を使うと速いが、ドリフトを見逃す。トレードオフを理解して使うこと。

#### 1-5. state から外す（実体は残る）

```bash
# state から外す = Terraform の管理下から外れるだけ。ファイルは消えない
terraform state rm local_file.greeting
terraform state list          # local_file.greeting が消えている
ls -l hello.txt               # ファイルは実在したまま！
terraform plan                # それでも「作る」と言われる
```

**ファイルが目の前に存在しているのに、Terraform は「作る」と言う。** ②（state）に無いものは、③（現実）にあっても Terraform には見えないからである。1.1 の図の赤枠の意味がここで実感できる。

```bash
# 元に戻す：もう一度 apply して管理下に置く
terraform apply -auto-approve
terraform state list
```

#### 1-6. `random_pet` には「実体」が存在しない

ここで、初学者がほぼ必ず引っかかる点を潰しておく。

`hello.txt` の中身が `hello, key-stud` だとして、**`random_pet.name` の実体はこの `key-stud` という文字列なのか？**

答えは **No**。`random_pet.name` の実体は **state の中だけ**にあり、外の世界には何も存在しない。`hello.txt` に書かれている `key-stud` は、`local_file` が `${random_pet.name.id}` を参照した**結果のコピー**であって、`random_pet` そのものではない。

実験で確かめる。

```bash
cat hello.txt                 # hello, key-stud
rm hello.txt                  # ファイルを消す
terraform apply -auto-approve
cat hello.txt                 # hello, key-stud  ← 名前が変わらない！
```

**ファイルを消して作り直しても、ペット名が同じまま**なのが決定的な証拠である。もし実体がファイルの中の文字列なら、消えた時点で失われて別の名前が生成されるはずだった。実際には **Terraform は state に記録された `key-stud` を読み出して、同じ内容のファイルを復元している**。

逆向きも見ておく。

```bash
terraform state rm random_pet.name    # ペットだけ管理から外す
cat hello.txt                          # hello, key-stud （ファイルは無傷）
terraform plan
```

```
  # local_file.greeting must be replaced
  # random_pet.name will be created
Plan: 2 to add, 0 to change, 1 to destroy.
```

**ファイルは無傷なのに、`local_file` まで作り直しになる。** 新しいペットが生成されれば `content` が変わるので、それに依存する `local_file` も巻き添えで replace される。**依存グラフが state 経由で連鎖している**のが見える瞬間である。

```bash
terraform apply -auto-approve
cat hello.txt                 # 今度は別の名前になっている
```

> **`random_*` は「state だけに存在するリソース」**である。1.1 の3つの世界でいうと、**③（現実）に対応物が無い**という特殊なケース。だから、
> - **ドリフトが原理的に起こらない**（比較すべき現実が無い）
> - **state を失うと、値が永久に失われる**（AWS に問い合わせて取り戻すことができない）
>
> **これはカタログ運用で実際に効く。** 演習2 で `random_id.suffix` を S3 バケット名に使うが、**state を失うとサフィックスが再生成され、Terraform は「別のバケット」を作りにいく**。既存バケットは孤児として残る。「state を失う＝管理権を失う」の最も分かりやすい実例がこれである。
>
> なお、意図的に再生成したいときは `keepers` を使う。
> ```hcl
> resource "random_pet" "name" {
>   keepers = {
>     # この値が変わったときだけ、新しいペット名を生成する
>     environment = var.environment
>   }
> }
> ```

#### 1-7. 強制的に作り直す

```bash
# 実体ごと作り直す（旧 taint）
terraform apply -replace=random_pet.name -auto-approve
cat hello.txt                 # 名前が変わっている
```

`-replace` の plan では、1.4 で見た **`-/+` (replace)** の記号が出ているはずである。

#### 1-8. 片付け

```bash
terraform destroy -auto-approve
ls hello.txt   # No such file
```

> **この演習で学んだこと**
> - `plan` = ①と②の差。`apply` = ③に反映して②を更新。`refresh` = ③を②に取り込む。
> - `terraform state rm` は**実体を消さない**。管理から外すだけ。逆に言えば、**state を失うと実体は残るが触れなくなる**。
> - **`random_*` は state だけに存在する**。③（現実）に対応物が無いので、state を失うと値は永久に取り戻せない。
> - 依存関係は参照から自動で作られる。`depends_on` は最後の手段。
> - `(known after apply)` は「まだ決まっていない」という意味であって、エラーではない。

---

### 演習2: AWS に1つだけ作り、plan を精読する

**目的**: 実際の AWS プロバイダに触れ、**`plan` の差分を読んで危険を検知する**訓練をする。

#### 2-1. 構成を書く

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/ex2 && cd /c/dev/handson-terraform-beginner/tf-handson/ex2
```

`main.tf`:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"

  # このプロバイダで作る全リソースに自動で付くタグ
  default_tags {
    tags = {
      ManagedBy = "terraform"
      Handson   = "tf-catalog"
    }
  }
}

resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "demo" {
  bucket = "tf-handson-demo-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "demo" {
  bucket = aws_s3_bucket.demo.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

> **つまずきポイント**: `aws_s3_bucket` に `versioning { ... }` を直接書けたのは AWS プロバイダ v3 までである。**v4 以降は versioning / encryption / public access block がすべて別リソースに分離された。** ネットの古い記事をコピペすると `Unsupported block type` で落ちるのはこれが原因。

#### 2-2. init して plan を精読する

```bash
terraform init
terraform plan
```

次の3点を必ず自分の目で確認する。

| 見るところ | 何を確認するか |
|-----------|--------------|
| 各リソースの先頭行 `# aws_s3_bucket.demo will be created` | **どのリソースが**どうなるか（サマリではなくここ） |
| 行頭の記号 | `+` / `~` / `-/+` / `-` |
| `(known after apply)` の箇所 | apply しないと決まらない値（ARN、ID など） |

#### 2-3. apply する

```bash
terraform apply
```

`yes` を手で打つ。CI 以外では `-auto-approve` を使わない癖をつけたほうがよい。

```bash
aws s3 ls | grep tf-handson-demo
```

#### 2-4. 「作り直し」が起きる変更を体験する

S3 のバケット名は**変更できない属性**である。名前を変えるとどうなるか見る。

```bash
sed -i 's/tf-handson-demo-/tf-handson-renamed-/' main.tf
terraform plan
```

```
  # aws_s3_bucket.demo must be replaced
-/+ resource "aws_s3_bucket" "demo" {
      ~ bucket = "tf-handson-demo-a1b2c3d4" -> "tf-handson-renamed-a1b2c3d4" # forces replacement
      ...
Plan: 1 to add, 1 to change, 1 to destroy.
```

**`# forces replacement` というコメントが、その属性が作り直しの原因であることを示している。** これが本番の RDS や EBS で出たら、データが消える。`plan` を読むというのは、この行を探す作業のことである。

適用せずに戻す。

```bash
sed -i 's/tf-handson-renamed-/tf-handson-demo-/' main.tf
terraform plan   # No changes
```

#### 2-5. 手動変更（ドリフト）を検知する

マネジメントコンソールでの手作業を CLI で模倣する。

```bash
BUCKET=$(aws s3 ls | grep tf-handson-demo | awk '{print $3}')
echo "$BUCKET"
aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Suspended

terraform plan
```

```
Note: Objects have changed outside of Terraform
  # aws_s3_bucket_versioning.demo has changed
  ~ versioning_configuration {
      ~ status = "Enabled" -> "Suspended"
    }
...
Plan: 0 to add, 1 to change, 0 to destroy.
```

**Terraform は「勝手に変えられた設定を元に戻す」方向に働く。** これがコード管理の価値であり、同時に「コンソールで直したのに翌日戻っている」という現象の正体でもある。

```bash
terraform apply -auto-approve   # Enabled に戻る
```

#### 2-6. 片付け（次の演習で作り直す）

```bash
terraform destroy -auto-approve
```

> **この演習で学んだこと**
> - AWS プロバイダ v4 以降、S3 の付帯設定は別リソース。古い記事に注意。
> - `default_tags` を使うと全リソースにタグが付く。カタログの必須タグ実装の土台になる。
> - `# forces replacement` こそが plan で最初に探すべき文字列。
> - ドリフトは `plan` が勝手に検知する。Terraform は「戻す」方向に働く。

---

### 演習3: 変数・出力・検証 — 「注文フォーム」を設計する

**目的**: ハードコードを排除し、**利用者に見せるインターフェース**を設計する。ここがカタログ設計の本体になる。

#### 3-1. 変数を切り出す

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/ex3 && cd /c/dev/handson-terraform-beginner/tf-handson/ex3
cp ../ex2/main.tf .
```

`variables.tf`:

```hcl
variable "region" {
  description = "リソースを作るリージョン"
  type        = string
  default     = "ap-northeast-1"
}

variable "environment" {
  description = "環境識別子"
  type        = string

  validation {
    condition     = contains(["dev", "stg", "prd"], var.environment)
    error_message = "environment は dev / stg / prd のいずれかを指定してください。"
  }
}

variable "owner_team" {
  description = "このリソースの持ち主チーム（必須タグになる）"
  type        = string

  validation {
    condition     = length(trimspace(var.owner_team)) > 0
    error_message = "owner_team は必須です。持ち主のいないリソースは作れません。"
  }
}

variable "retention_days" {
  description = "古いバージョンを保持する日数"
  type        = number
  default     = 90

  validation {
    condition     = var.retention_days >= 7 && var.retention_days <= 3650
    error_message = "retention_days は 7〜3650 の範囲で指定してください。"
  }
}

variable "additional_tags" {
  description = "追加タグ"
  type        = map(string)
  default     = {}
}
```

`main.tf` の `provider` と `resource` を書き換える。

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = merge(
      {
        ManagedBy   = "terraform"
        Environment = var.environment
        OwnerTeam   = var.owner_team
      },
      var.additional_tags
    )
  }
}

resource "aws_s3_bucket" "demo" {
  bucket = "tf-handson-${var.environment}-${random_id.suffix.hex}"
}
```

`outputs.tf`:

```hcl
output "bucket_name" {
  description = "作成されたバケット名"
  value       = aws_s3_bucket.demo.id
}

output "bucket_arn" {
  description = "作成されたバケットの ARN"
  value       = aws_s3_bucket.demo.arn
}
```

#### 3-2. validation が効くことを確認する

```bash
terraform init
terraform plan -var 'environment=production' -var 'owner_team=team-alpha'
```

```
Error: Invalid value for variable

  on variables.tf line 8:
   8: variable "environment" {
    ├────────────────
    │ var.environment is "production"

environment は dev / stg / prd のいずれかを指定してください。

This was checked by the validation rule at variables.tf:12,3-13.
```

**AWS のリソースには一切アクセスせず、plan の入り口で弾かれた。** これがカタログのガードレールの正体である。Wiki に書いた規約と違い、これは **回避できない**。

正しい値なら通る。

```bash
terraform plan -var 'environment=dev' -var 'owner_team=team-alpha'
```

#### 3-3. tfvars ファイルを使う

`terraform.tfvars`:

```hcl
environment = "dev"
owner_team  = "team-alpha"

additional_tags = {
  CostCenter = "CC-1234"
}
```

```bash
terraform plan     # -var なしで通る
terraform apply -auto-approve
terraform output
terraform output -raw bucket_name
terraform output -json | jq
```

#### 3-4. 変数の優先順位を知る

同じ変数が複数の場所で指定された場合、**後にあるものが勝つ**。

| 優先度 | 指定方法 | 備考 |
|:---:|---------|------|
| 1（弱） | `variable` の `default` | — |
| 2 | 環境変数 `TF_VAR_environment` | CI で便利 |
| 3 | `terraform.tfvars` | 自動で読まれる |
| 4 | `terraform.tfvars.json` | 自動で読まれる |
| 5 | `*.auto.tfvars` / `*.auto.tfvars.json` | ファイル名のアルファベット順 |
| 6（強） | `-var` / `-var-file` | **コマンドラインの記述順**で後勝ち |

```bash
# 環境変数より -var が強いことを確認
export TF_VAR_environment=stg
terraform plan | head -5           # stg として扱われる
terraform plan -var environment=dev | head -5   # dev が勝つ
unset TF_VAR_environment
```

> **カタログ設計への含意**: 利用者に触らせるのは **`terraform.tfvars` 相当のファイルだけ**にし、`-var` は CI が使う、という分離ができる。演習9でこれを使う。

#### 3-5. `sensitive` を試す

**まず `outputs.tf` に追記する。** ここを飛ばして下の `terraform output` を実行すると `Error: Output "secret_example" not found` になる。

```bash
cat >> outputs.tf <<'EOF'

output "secret_example" {
  value     = "this-should-not-be-printed"
  sensitive = true
}
EOF
```

**追記したら apply する。** 出力（`output`）は state に保存される値なので、**`apply` を通さないと `terraform output` から読めない**。リソースが1つも変わらなくても apply は必要である。

```bash
terraform apply -auto-approve
```

```
Changes to Outputs:
  + secret_example = (sensitive value)

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

bucket_arn     = "arn:aws:s3:::tf-handson-dev-f25ad96b"
bucket_name    = "tf-handson-dev-f25ad96b"
secret_example = <sensitive>
```

> `Resources: 0 added, 0 changed, 0 destroyed.` なのに `Changes to Outputs:` が出ているのがポイント。**出力の追加は「インフラの変更ゼロ、state の変更あり」という操作**である。

ここから、`sensitive` がどこまで守ってくれるのかを確かめる。

```bash
terraform output                        # 一覧
terraform output secret_example         # 名指し
terraform output -raw secret_example    # 名指し + 生の文字列
terraform output -json                  # JSON
```

```
# terraform output
bucket_name    = "tf-handson-dev-f25ad96b"
secret_example = <sensitive>                      ← 隠れる

# terraform output secret_example
"this-should-not-be-printed"                      ← 出る！

# terraform output -raw secret_example
this-should-not-be-printed                        ← 出る

# terraform output -json
{
  "secret_example": {
    "sensitive": true,
    "type": "string",
    "value": "this-should-not-be-printed"         ← 出る
  }
}
```

> **`sensitive` の守備範囲は、思っているよりずっと狭い**
>
> | 操作 | 隠れるか |
> |------|:---:|
> | `terraform plan` / `apply` のログ | ✅ 隠れる（`(sensitive value)`） |
> | `terraform output`（一覧） | ✅ 隠れる（`<sensitive>`） |
> | **`terraform output <名前>`（名指し）** | ❌ **出る** |
> | **`terraform output -json`** | ❌ **出る**（`"sensitive": true` のフラグが付くだけ） |
> | **state ファイルの中身** | ❌ **平文で保存される** |
> | CI のジョブログ | 上記に準ずる（`-json` を使えば漏れる） |
>
> つまり `sensitive` は **「うっかり画面やログに出さないための目隠し」であって、アクセス制御ではない**。名指しで聞けば誰にでも答える。
>
> **カタログ運用への含意**: パスワードやトークンを Terraform で扱うなら、守るべきは出力ではなく **state そのもの**である。
> - state バケットの読み取り権限を絞る（演習4で暗号化と公開ブロックを入れる理由）
> - CI の plan アーティファクトを Guest に見せない（演習8の `artifacts:access: developer`）
> - そもそも**秘密を Terraform で生成・保持しない**（Secrets Manager 等に置き、Terraform は ARN だけを扱う）が最も筋がよい

#### 3-6. 片付け（今回はしない）

> **ここでは `destroy` しない。** 次の演習4 は、**このディレクトリのローカル state をリモートへ移行する**のが題材なので、いま作ったリソースと state をそのまま使う。
>
> 演習4 を後日にする場合だけ、次で片付けてよい。その場合は演習4 の冒頭で `terraform apply -auto-approve` を実行し、リソースがある状態に戻してから始めること。
>
> ```bash
> terraform destroy -auto-approve
> ```

状態を確認して次へ進む。

```bash
terraform state list       # 3つのリソースが並んでいる
ls terraform.tfstate       # ローカルに state がある（演習4 でこれを移行する）
```

> **この演習で学んだこと**
> - `validation` はクラウドに触れる前に弾く、最も安いガードレール。
> - 変数の優先順位は6段階。CI では環境変数、利用者には tfvars、という分離ができる。
> - **`sensitive` はアクセス制御ではない。** 一覧では隠れるが名指しすれば出る。守るべきは state そのもの。
> - 出力の追加は「インフラ変更ゼロ、state 変更あり」。`apply` しないと `terraform output` から読めない。

---

### 演習4: state をリモートへ — チームで使える形にする

**目的**: ローカルの `terraform.tfstate` を捨て、**S3 バックエンド + ネイティブロック**に移行する。ここを越えないとチーム運用は始まらない。

#### 4-1. なぜローカル state ではダメなのか

| 問題 | 起きること |
|------|----------|
| 共有できない | 他の人の `plan` が「全部作る」と言い出す |
| 同時実行を防げない | 2人が同時に apply して state が壊れる |
| バックアップが無い | PC が壊れると管理権を失う |
| Git に入れると事故る | **state には平文の機密が入っている**（3-5 参照） |

#### 4-2. state 置き場を先に作る（鶏と卵の解決）

state 置き場自体は Terraform で作るとブートストラップ問題が起きる。**最初の1回だけ CLI で作る**のが最も素直である。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
STATE_BUCKET="tfstate-handson-${ACCOUNT_ID}"
REGION=ap-northeast-1

aws s3api create-bucket \
  --bucket "$STATE_BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"

# state の世代管理（壊したときの命綱）
aws s3api put-bucket-versioning \
  --bucket "$STATE_BUCKET" \
  --versioning-configuration Status=Enabled

# 暗号化
aws s3api put-bucket-encryption \
  --bucket "$STATE_BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# 公開を完全に塞ぐ
aws s3api put-public-access-block \
  --bucket "$STATE_BUCKET" \
  --public-access-block-configuration \
  'BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true'

echo "STATE_BUCKET=$STATE_BUCKET"
```

> **バージョニングは必須**。state を壊したときに、S3 の以前のバージョンを取り出して復旧できるかどうかが分かれ目になる。

#### 4-3. backend を宣言して移行する

演習3のディレクトリで作業する。

```bash
cd /c/dev/handson-terraform-beginner/tf-handson/ex3
terraform state list               # 演習3 のリソースがそのまま残っているはず
```

演習3 の最後で `destroy` してしまった場合は、ここで作り直してから進む。

```bash
terraform apply -auto-approve      # ローカル state のある状態を作る
```

`backend.tf`（**バケット名は自分の値に置き換える**）:

```hcl
terraform {
  backend "s3" {
    bucket       = "tfstate-handson-123456789012"
    key          = "handson/ex3/terraform.tfstate"
    region       = "ap-northeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

```bash
terraform init -migrate-state
```

```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. ...
  Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes
```

移行できたことを確認する。

```bash
aws s3 ls "s3://${STATE_BUCKET}/handson/ex3/"
ls -la terraform.tfstate*      # ローカルは terraform.tfstate.backup だけ残る
terraform state list           # リモートから読めている
```

> **`use_lockfile = true` について**: Terraform 1.10 で導入され 1.11 で正式化された **S3 ネイティブロック**。ロック中は `key` と同じ場所に `terraform.tfstate.tflock` というオブジェクトが作られる。
> **従来必要だった DynamoDB テーブル（`dynamodb_table`）は非推奨**となり、将来のマイナーバージョンで削除される予定。既存環境からの移行時は両方を同時に指定できる。**新規で DynamoDB を作る必要はもう無い。**

#### 4-4. ロックが効くことを目で見る

**ターミナルを2つ**開いて確認する。

ターミナル1:

```bash
cd /c/dev/handson-terraform-beginner/tf-handson/ex3
terraform apply           # yes を打たずに、確認プロンプトで止めておく
```

ターミナル2（同時に）:

```bash
cd /c/dev/handson-terraform-beginner/tf-handson/ex3
terraform plan
```

```
│ Error: Error acquiring the state lock
│
│ Error message: operation error S3: PutObject, ... PreconditionFailed
│ Lock Info:
│   ID:        1a2b3c4d-...
│   Path:      tfstate-handson-.../handson/ex3/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       you@your-pc
│   Created:   2026-09-05 ...
```

ロック中のオブジェクトも見える。

```bash
aws s3 ls "s3://${STATE_BUCKET}/handson/ex3/"
# terraform.tfstate
# terraform.tfstate.tflock   ← これ
```

ターミナル1 で `no` を入力して中断すると、ロックが解放される。

> **もし異常終了でロックが残ったら**: エラーに表示された `ID` を使って `terraform force-unlock <ID>` を実行する。**必ず「本当に他の誰も実行していない」ことを確認してから**行うこと。生き残っている apply を横から解除すると state が壊れる。

#### 4-5. state を消してみる（そして復旧する）

**この演習の山場。** state を失うとどうなるかを、安全な場所で一度だけ体験しておく価値がある。

```bash
# 現在の state を退避（保険）
aws s3 cp "s3://${STATE_BUCKET}/handson/ex3/terraform.tfstate" ./state-backup.json

# state を消す
aws s3 rm "s3://${STATE_BUCKET}/handson/ex3/terraform.tfstate"

terraform plan
```

```
Plan: 4 to add, 0 to change, 0 to destroy.
```

**AWS 上にバケットは実在しているのに、Terraform は「全部作る」と言っている。** ここで apply すると `BucketAlreadyOwnedByYou` で落ちる。これが「state を失う」という事故の中身である。

復旧する。

```bash
aws s3 cp ./state-backup.json "s3://${STATE_BUCKET}/handson/ex3/terraform.tfstate"
terraform plan     # No changes
rm state-backup.json
```

> **実務での復旧手段**: 4-2 でバージョニングを有効にしたので、実際には `aws s3api list-object-versions` で以前の state を取り出せる。**バージョニングを付けていないと、この復旧はできない。**

#### 4-6. 片付け

```bash
terraform destroy -auto-approve
```

> **この演習で学んだこと**
> - backend の移行は `terraform init -migrate-state` の一発。
> - **`use_lockfile = true` が現在の標準。DynamoDB はもう不要。**
> - ロックは `.tflock` オブジェクトとして S3 上に見える。
> - state バケットには**バージョニングと暗号化と公開ブロックを必ず入れる**。

---

### 演習5: モジュール化 — カタログの「商品」を作る

**目的**: 演習3の構成を、**社内の誰が使っても安全な部品**に作り替える。ここからが本題である。

#### 5-1. 何を固定し、何を選ばせるかを決める

商品設計は Terraform を書く前に決める。

| 項目 | 誰が決めるか | 理由 |
|------|------------|------|
| 暗号化する／しない | **モジュールが固定（必ずする）** | 選ばせる理由が無い |
| パブリックアクセスブロック | **モジュールが固定（必ず全部ブロック）** | 事故の最大要因 |
| バケット命名規則 | **モジュールが生成** | 一意性と検索性を担保 |
| 必須タグ（Owner / CostCenter / Env） | **利用者が入力（必須）** | 中央では知り得ない |
| バージョニング | 利用者が選択（既定 ON） | 用途で変わる |
| 保持日数 | 利用者が選択（範囲を制限） | 用途で変わる |
| `force_destroy` | 利用者が選択（**prd では禁止**） | 事故防止 |

> **カタログ設計の原則**: 「選ばせるほど親切」ではない。**選択肢は、意味のある差が出る項目にだけ与える。** 選択肢が多いモジュールは、レビューできないモジュールになる。

#### 5-2. ディレクトリを作る

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/tf-module-s3-bucket && cd /c/dev/handson-terraform-beginner/tf-handson/tf-module-s3-bucket
```

`versions.tf`:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source = "hashicorp/aws"
      # モジュール側は「動く範囲」を広めに宣言する。
      # 具体的なバージョン固定は呼び出し側（ルート）の責務。
      version = ">= 5.0, < 7.0"
    }
  }
}
```

> **最重要の作法**: **モジュールの中に `provider "aws" { ... }` ブロックを書いてはいけない。**
> モジュールは呼び出し側のプロバイダ設定を継承する。モジュール内に provider を書くと、
> - `for_each` / `count` をそのモジュールに使えなくなる
> - そのモジュールを構成から**削除できなくなる**（削除するには一度 provider を残したまま destroy する必要がある）
> 書いてよいのは `required_providers` だけである。

`variables.tf`:

```hcl
variable "org_prefix" {
  description = "組織を表す短い接頭辞（バケット名の先頭に付く）"
  type        = string
  default     = "acme"
}

variable "name" {
  description = "バケットの用途を表す短い名前（例: logs, artifacts）"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,20}$", var.name))
    error_message = "name は小文字英数字とハイフンのみ、3〜21文字、先頭は英字にしてください。"
  }
}

variable "environment" {
  description = "環境識別子"
  type        = string

  validation {
    condition     = contains(["dev", "stg", "prd"], var.environment)
    error_message = "environment は dev / stg / prd のいずれかです。"
  }
}

variable "owner_team" {
  description = "持ち主チーム（必須タグ）"
  type        = string

  validation {
    condition     = can(regex("^team-[a-z0-9-]+$", var.owner_team))
    error_message = "owner_team は team-xxx の形式で指定してください。"
  }
}

variable "cost_center" {
  description = "コストセンター（必須タグ）"
  type        = string

  validation {
    condition     = can(regex("^CC-[0-9]{4}$", var.cost_center))
    error_message = "cost_center は CC-1234 の形式で指定してください。"
  }
}

variable "versioning_enabled" {
  description = "オブジェクトのバージョニングを有効にするか"
  type        = bool
  default     = true
}

variable "retention_days" {
  description = "非現行バージョンを保持する日数"
  type        = number
  default     = 90

  validation {
    condition     = var.retention_days >= 7 && var.retention_days <= 3650
    error_message = "retention_days は 7〜3650 の範囲で指定してください。"
  }
}

variable "force_destroy" {
  description = "中身が入っていてもバケットを削除できるようにするか"
  type        = bool
  default     = false

  # Terraform 1.9 以降、validation から他の変数を参照できる
  validation {
    condition     = !(var.force_destroy && var.environment == "prd")
    error_message = "prd 環境では force_destroy を有効にできません。"
  }
}

variable "additional_tags" {
  description = "追加タグ"
  type        = map(string)
  default     = {}
}
```

> **知っておくと得をする変更点**: **Terraform 1.9 以降、`validation` の `condition` から他の変数を参照できるようになった。** 「`prd` のときだけ `force_destroy` を禁止する」のような**組み合わせのガードレール**が、外部ツールなしで書ける。1.8 以前を前提にした記事では「できない」と書かれているので注意。

`main.tf`:

```hcl
data "aws_caller_identity" "current" {}

locals {
  bucket_name = "${var.org_prefix}-${var.environment}-${var.name}-${data.aws_caller_identity.current.account_id}"

  required_tags = {
    ManagedBy   = "terraform"
    CatalogItem = "s3-bucket"
    Environment = var.environment
    OwnerTeam   = var.owner_team
    CostCenter  = var.cost_center
  }
}

resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy

  tags = merge(local.required_tags, var.additional_tags)
}

# --- ここから下は利用者が選べない（＝ガードレール） ---

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  # バージョニング設定より後に適用されないと警告が出る
  depends_on = [aws_s3_bucket_versioning.this]

  rule {
    id     = "expire-noncurrent-versions"
    status = "Enabled"

    # v4 以降、rule には filter か prefix が必須。空の filter = 全オブジェクト
    filter {}

    noncurrent_version_expiration {
      noncurrent_days = var.retention_days
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

`outputs.tf`:

```hcl
output "bucket_id" {
  description = "バケット名（ID）"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "バケットの ARN"
  value       = aws_s3_bucket.this.arn
}

output "bucket_regional_domain_name" {
  description = "リージョン付きドメイン名"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
```

`README.md`（**Module Registry の画面に表示される**ので手を抜かない）:

````markdown
# s3-bucket

社内標準の S3 バケットを1つ払い出すモジュール。

暗号化・パブリックアクセスブロック・ライフサイクルは固定で有効になる。

## 使い方

```hcl
module "logs" {
  source  = "gitlab.com/<YOUR-GROUP>/s3-bucket/aws"
  version = "0.1.0"

  name        = "logs"
  environment = "dev"
  owner_team  = "team-alpha"
  cost_center = "CC-1234"
}
```

## 入力

| 名前 | 型 | 既定値 | 必須 | 説明 |
|------|-----|-------|:---:|------|
| name | string | — | ✅ | 用途を表す短い名前（3〜21文字、小文字英数字とハイフン） |
| environment | string | — | ✅ | `dev` / `stg` / `prd` |
| owner_team | string | — | ✅ | `team-xxx` 形式 |
| cost_center | string | — | ✅ | `CC-1234` 形式 |
| versioning_enabled | bool | `true` | | バージョニング |
| retention_days | number | `90` | | 非現行バージョンの保持日数（7〜3650） |
| force_destroy | bool | `false` | | `prd` では指定不可 |
| additional_tags | map(string) | `{}` | | 追加タグ |

## 出力

| 名前 | 説明 |
|------|------|
| bucket_id | バケット名 |
| bucket_arn | ARN |
| bucket_regional_domain_name | リージョン付きドメイン名 |
````

#### 5-3. ローカルから呼んで動作確認する

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/ex5 && cd /c/dev/handson-terraform-beginner/tf-handson/ex5
```

`main.tf`:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"   # 呼び出し側で固定する
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

module "logs" {
  source = "../tf-module-s3-bucket"

  name        = "logs"
  environment = "dev"
  owner_team  = "team-alpha"
  cost_center = "CC-1234"

  retention_days = 30
}

output "logs_bucket" {
  value = module.logs.bucket_id
}
```

```bash
terraform init
terraform plan
terraform apply -auto-approve
terraform output
```

#### 5-4. ガードレールが効くことを確認する

```bash
# 命名規約違反
sed -i 's/name        = "logs"/name        = "MyLogs_2024"/' main.tf
terraform plan
```

```
Error: Invalid value for variable

  on main.tf line 20, in module "logs":
  20:   name        = "MyLogs_2024"
    ├────────────────
    │ var.name is "MyLogs_2024"

name は小文字英数字とハイフンのみ、3〜21文字、先頭は英字にしてください。

This was checked by the validation rule at
../tf-module-s3-bucket/variables.tf:11,3-13.
```

```bash
sed -i 's/name        = "MyLogs_2024"/name        = "logs"/' main.tf

# 環境と force_destroy の組み合わせ違反
sed -i 's/environment = "dev"/environment = "prd"/' main.tf
sed -i '/retention_days = 30/a\  force_destroy  = true' main.tf
terraform plan
```

```
Error: Invalid value for variable

  on main.tf line 26, in module "logs":
  26:   force_destroy  = true
    ├────────────────
    │ var.environment is "prd"
    │ var.force_destroy is true

prd 環境では force_destroy を有効にできません。

This was checked by the validation rule at
../tf-module-s3-bucket/variables.tf:70,3-13.
```

**利用者がどれだけ頑張っても、規約違反のリソースは作れない。** これが「カタログの商品」であることの意味である。

```bash
# 元に戻す
sed -i 's/environment = "prd"/environment = "dev"/' main.tf
sed -i '/force_destroy  = true/d' main.tf
terraform plan   # 通る
```

#### 5-5. 片付け

```bash
terraform destroy -auto-approve
```

> **この演習で学んだこと**
> - **モジュールに `provider` ブロックを書かない**。`required_providers` だけ書く。
> - バージョン制約は「モジュールは緩く、ルートは固く」。
> - ガードレールは `validation` と「そもそも変数にしない」の2本立て。
> - Terraform 1.9+ なら変数間の相互 validation が書ける。
> - `README.md` は Module Registry のカタログ画面に出る＝商品説明書である。

---

### 演習6: GitLab の Terraform Module Registry に publish する

**目的**: 作った商品を、**バージョン付きで社内の棚に並べる**。

#### 6-1. GitLab 側の準備

1. **トップレベルグループを1つ作る**（例: `acme-platform`）
   - Module Registry の名前空間は**トップレベルグループ**である。サブグループではない。
   - **グループ名・プロジェクト名にドット `.` を含めてはいけない**（`source` の解決が壊れる）。
2. そのグループの下にプロジェクト `s3-bucket` を作る。

> **命名の制約（ハマりどころ）**
> | 対象 | 制約 |
> |------|------|
> | モジュール名 | 1〜64文字、**小文字英数字のみ**。アンダースコアはハイフンに変換される |
> | モジュールシステム | 同上（`aws` / `google` / `local` など） |
> | グループ名・プロジェクト名 | **ドットを含めない** |
> | バージョン | **セマンティックバージョニング**。`v` 接頭辞を付けない（`0.1.0` であって `v0.1.0` ではない） |

#### 6-2. モジュールを push する

```bash
cd /c/dev/handson-terraform-beginner/tf-handson/tf-module-s3-bucket
git init -b main
cp ../\.gitattributes . 2>/dev/null || true

cat > .gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.*
crash.log
EOF

git add -A
git commit -m "feat: 社内標準 S3 バケットモジュールの初版"
git remote add origin https://gitlab.com/acme-platform/s3-bucket.git
git push -u origin main
```

#### 6-3. publish 用の CI を置く

`.gitlab-ci.yml`:

```yaml
include:
  - template: Terraform-Module.gitlab-ci.yml

variables:
  TERRAFORM_MODULE_DIR: ${CI_PROJECT_DIR}
  TERRAFORM_MODULE_NAME: s3-bucket   # 小文字英数字のみ
  TERRAFORM_MODULE_SYSTEM: aws
  TERRAFORM_MODULE_VERSION: ${CI_COMMIT_TAG}
```

このテンプレートは3つのジョブを持つ。

| ジョブ | 内容 | 実行タイミング |
|--------|------|--------------|
| `fmt` | `terraform fmt -check` | 全パイプライン |
| `kics-iac-sast` | IaC の静的セキュリティスキャン | 全パイプライン |
| `deploy` | Module Registry に publish | **タグが付いたときだけ** |

テンプレートを使わず自前で書くこともできる（自己管理インスタンスでテンプレートを使えない場合など）。

```yaml
stages: [deploy]

upload:
  stage: deploy
  image: curlimages/curl:latest
  variables:
    TERRAFORM_MODULE_DIR: ${CI_PROJECT_DIR}
    TERRAFORM_MODULE_NAME: s3-bucket
    TERRAFORM_MODULE_SYSTEM: aws
    TERRAFORM_MODULE_VERSION: ${CI_COMMIT_TAG}
  script:
    - TGZ="/tmp/${TERRAFORM_MODULE_NAME}-${TERRAFORM_MODULE_SYSTEM}-${TERRAFORM_MODULE_VERSION}.tgz"
    - tar -vczf "$TGZ" -C "${TERRAFORM_MODULE_DIR}" --exclude=./.git .
    - 'curl --fail-with-body --location --header "JOB-TOKEN: ${CI_JOB_TOKEN}"
         --upload-file "$TGZ"
         "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/terraform/modules/${TERRAFORM_MODULE_NAME}/${TERRAFORM_MODULE_SYSTEM}/${TERRAFORM_MODULE_VERSION}/file"'
  rules:
    - if: $CI_COMMIT_TAG
```

#### 6-4. タグを打って publish する

```bash
git add .gitlab-ci.yml
git commit -m "ci: Module Registry への publish を追加"
git push

# v を付けない！
git tag 0.1.0
git push origin 0.1.0
```

GitLab の **CI/CD > パイプライン** で `deploy` ジョブが成功したら、**操作（Operate）> Terraform モジュール** を開く。`s3-bucket/aws` が `0.1.0` として1件並んでいれば成功である。

一覧に出たら、モジュール名をクリックして詳細画面に入り、**`README` タブに 5-2 で書いた商品説明書が表示されている**ことを確認する。ここが利用者から見える唯一の説明になる。

> **⚠ ここで自分の名前空間を控えておく（以降の演習で必須）**
>
> 本教材は名前空間を `acme-platform` というプレースホルダで書いている。**演習7・8・9 の `source` は、すべて自分のトップレベルグループ名に読み替える必要がある。**
>
> ```hcl
> # 教材の表記
> source  = "gitlab.com/acme-platform/s3-bucket/aws"
>
> # 自分の環境（例: トップレベルグループが handson-idp-catalog の場合）
> source  = "gitlab.com/handson-idp-catalog/s3-bucket/aws"
> ```
>
> 名前空間はレジストリ画面のパンくず（`<グループ名> / s3-bucket / Terraformモジュールレジストリ`）の左端、または `Operate > Terraform モジュール` のグループページの URL で確認できる。**サブグループではなくトップレベルグループ**である点に注意。
>
> 読み替え漏れがあると、演習7 の `terraform init` で次のように失敗する。
>
> ```
> Error: Failed to retrieve available versions for module "logs"
> ...could not read module registry: 404 Not Found
> ```

> **`v0.1.0` と打ってしまった場合**: publish 自体は通ることがあるが、`version = "0.1.0"` での解決に失敗する。タグを打ち直す（`git tag -d v0.1.0` → `git push --delete origin v0.1.0` → `git tag 0.1.0`）。**発行済みモジュールは編集できない**ので、Registry 側からも削除して再発行する。

> **この演習で学んだこと**
> - Module Registry の名前空間は**トップレベルグループ**。
> - 発行のトリガーは **Git タグ**。バージョンは **`v` 無しの semver**。
> - `README.md` は商品説明書としてカタログ画面に出る。
> - 発行済みバージョンは**上書きできない**（削除して再発行するしかない）。だから semver を守る意味がある。

---

### 演習7: 利用者側から module を呼ぶ

**目的**: Registry 経由でモジュールを取得し、**ローカルパス参照との違い**を理解する。

#### 7-1. 認証を通す

Registry からの取得には認証が要る。**アクセストークンを1つ作る。**

1. GitLab で **個人アクセストークン**（スコープ: **`read_api`**）またはグループの**デプロイトークン**（スコープ: **`read_package_registry`**）を発行する。
   > **スコープの選択を間違えると 403 になる。** 名前が似ていて紛らわしいものが2つある。
   > | スコープ | 用途 | Terraform モジュールに効くか |
   > |---|---|:---:|
   > | `read_api` | API 全体の読み取り | ✅ **これが正解** |
   > | `read_repository` | リポジトリの clone | ❌ |
   > | `read_registry` | **コンテナ**レジストリ | ❌ |
2. 環境変数に入れる。**変数名はホスト名のドットをアンダースコアに置き換えた形式**。

```bash
# GitBash（このシェルでのみ有効）
export TF_TOKEN_gitlab_com='glpat-xxxxxxxxxxxxxxxxxxxx'
```

**`terraform init` を打つ前に、トークンが通ることを確認しておく**と切り分けが一瞬で済む（`<YOUR-GROUP>` は自分のトップレベルグループ名）。

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}
' -H "Authorization: Bearer ${TF_TOKEN_gitlab_com}"   "https://gitlab.com/api/v4/packages/terraform/modules/v1/<YOUR-GROUP>/s3-bucket/aws/versions"
```

| 返り値 | 意味 |
|:---:|---|
| **200** | 正常。`terraform init` に進んでよい |
| **401** | トークンが送られていない（変数名のミス、別シェルで export した、値が空） |
| **403** | トークンは届いている。原因は2つ。**① 名前空間（グループパス）の指定間違い**、**② スコープ不足**（`read_api` になっていない） |
| **404** | パスの形自体が不正（セグメント数が違うなど） |

> **403 が出たら、まずスコープではなくパスを疑う。** GitLab は「存在しない／アクセス権のない名前空間」に対して 404 ではなく **403** を返す。リソースの存在有無を外部に漏らさないための挙動である。
> レジストリ画面のパンくずに出ているグループ名を、**コピー＆ペーストで**貼り直すのが確実（目で写すと `-` と `_`、単数形と複数形で間違えやすい）。

理由の本文を読みたいときは `-o /dev/null -w` を外して実行する。

自己管理インスタンス `gitlab.example.com` なら `TF_TOKEN_gitlab_example_com` になる。

恒久設定にするなら `~/.terraformrc`（Windows なら `%APPDATA%/terraform.rc`）に書く。

```hcl
credentials "gitlab.com" {
  token = "glpat-xxxxxxxxxxxxxxxxxxxx"
}
```

> **Windows のつまずき**: `export` は GitBash のそのセッション限り。PowerShell で作業するなら `$env:TF_TOKEN_gitlab_com = 'glpat-...'`。**環境変数名にドットは使えない**ので、この置換ルールを忘れると「なぜか 401」になる。

#### 7-2. Registry 経由で呼ぶ

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/ex7 && cd /c/dev/handson-terraform-beginner/tf-handson/ex7
```

`main.tf`:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

module "logs" {
  # ホスト名 / 名前空間（トップレベルグループ） / モジュール名 / システム
  source  = "gitlab.com/acme-platform/s3-bucket/aws"
  version = "0.1.0"

  name        = "logs"
  environment = "dev"
  owner_team  = "team-alpha"
  cost_center = "CC-1234"
}

output "logs_bucket" {
  value = module.logs.bucket_id
}
```

```bash
terraform init
```

```
Initializing modules...
Downloading gitlab.com/acme-platform/s3-bucket/aws 0.1.0 for logs...
- logs in .terraform/modules/logs
```

```bash
terraform plan
terraform apply -auto-approve
```

#### 7-3. `source` に変数は使えない、を確認する

```bash
# あえて壊してみる
cp main.tf main.tf.bak
sed -i 's|source  = "gitlab.com/acme-platform/s3-bucket/aws"|source  = "gitlab.com/${var.namespace}/s3-bucket/aws"|' main.tf
terraform init
```

```
│ Error: Variables not allowed
│ Variables may not be used here.
```

**`source` と `version` はリテラル文字列でなければならない。** モジュールの解決は変数評価より前に行われるため。「環境ごとに source を切り替える」という設計は取れないので、**ディレクトリを分けるか、`.tf` を生成する**しかない。

```bash
mv main.tf.bak main.tf
terraform init
```

#### 7-4. プロジェクト単位の参照も知っておく

グループ名前空間経由が使えない場合（重複許可、権限の都合など）は、プロジェクト直指定もできる。

```hcl
module "logs" {
  source = "https://gitlab.com/api/v4/projects/<PROJECT_ID>/packages/terraform/modules/s3-bucket/aws/0.1.0"
  # ...
}
```

この形式は `.netrc` による認証を使う。

```
machine gitlab.com
login <USERNAME>
password <TOKEN>
```

| | 名前空間参照 | プロジェクト参照 |
|---|------------|----------------|
| 書式 | `gitlab.com/<group>/<name>/<system>` | `https://gitlab.com/api/v4/projects/<id>/...` |
| `version` 引数 | **使える** | 使えない（URL に埋める） |
| 認証 | `TF_TOKEN_gitlab_com` | `.netrc` |
| 重複時 | 最後に publish されたものが勝つ | 明示的に一意 |

**通常は名前空間参照を使う。** `version` 引数が使えて、`~> 0.1` のような制約が書けるのが決定的な差である。

#### 7-5. 片付け

```bash
terraform destroy -auto-approve
```

> **この演習で学んだこと**
> - 認証は `TF_TOKEN_<host with underscores>` が最も扱いやすい。
> - **`source` / `version` に変数は使えない。**
> - `.terraform/modules/` に実体がダウンロードされている。ここを見れば「実際に何を使っているか」が分かる。

---

### 演習8: GitLab CI で「MR なら plan、merge なら apply」

**目的**: 人の手による `apply` をなくし、**レビュー可能な変更プロセス**にする。あわせて **AWS の長期キーを1つも保存しない**構成にする。

#### 8-1. AWS 側に OIDC 連携を作る

GitLab の CI ジョブが、アクセスキーなしで AWS の一時認証を得られるようにする。

**(a) ID プロバイダを登録する**

```bash
aws iam create-open-id-connect-provider \
  --url "https://gitlab.com" \
  --client-id-list "https://gitlab.com"
```

> AWS は主要な OIDC プロバイダについて自前の信頼ストアで証明書を検証するため、通常 `--thumbprint-list` は不要である。もし必須だと言われた場合は任意の40桁16進値を渡してよい（検証には使われない）。

**(b) 引き受けるロールを作る**

`trust-policy.json`（`<ACCOUNT_ID>` / グループ / プロジェクト名を自分の値に置き換える）:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/gitlab.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "gitlab.com:aud": "https://gitlab.com"
        },
        "StringLike": {
          "gitlab.com:sub": "project_path:acme-platform/tf-catalog:ref_type:branch:ref:*"
        }
      }
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name gitlab-tf-catalog \
  --assume-role-policy-document file://trust-policy.json

# 演習用。実務では S3 とタグ操作に絞った最小権限のカスタムポリシーにする
aws iam attach-role-policy \
  --role-name gitlab-tf-catalog \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam get-role --role-name gitlab-tf-catalog --query 'Role.Arn' --output text
```

> **`sub` クレームの形式**: `project_path:<group>/<project>:ref_type:<branch|tag>:ref:<name>`。
> **GitLab.com なら、さらに `gitlab.com:project_id` と `gitlab.com:namespace_id` を条件に使える。** これらはグループ名やプロジェクト名を変えても不変なので、**実務ではパスではなく ID で縛るほうが安全**である。自己管理インスタンスでは `sub` と `aud` しか使えない。

> **`aud` の注意**: ID プロバイダ登録時の `--client-id-list` と、CI 側の `id_tokens.aud` と、信頼ポリシーの `gitlab.com:aud` の**3つを一致させる**必要がある。ここがズレると `InvalidIdentityToken` で落ちる。

#### 8-2. カタログリポジトリを作る

GitLab に `acme-platform/tf-catalog` プロジェクトを作り、CI/CD 変数を1つ登録する。

| 変数名 | 値 | 種別 |
|--------|-----|------|
| `AWS_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/gitlab-tf-catalog` | 変数（マスクなしで可） |

登録時の設定で3箇所つまずきやすい。

| 設定 | どうすべきか | 間違えたときの症状 |
|------|------------|------------------|
| **キー** | `AWS_ROLE_ARN` 丁度（末尾スペースに注意） | `Error: role ARN is not set` |
| **変数を保護する** | 保護ブランチ以外でも `plan` を回すなら**オフ** | MR パイプラインだけ失敗する |
| **環境スコープ** | **`*`（All）** | `environment:` を持たない `plan` / `validate` だけ失敗する |

> **環境スコープは特に嵌まりやすい。** 本教材の `.gitlab-ci.yml` では `apply` ジョブだけが `environment: name: dev` を持つ。変数のスコープを `dev` に絞ると、**`apply` は通るのに `plan` が落ちる**という分かりにくい失敗になる。

値は手元で確認できる。

```bash
aws iam get-role --role-name gitlab-tf-catalog --query 'Role.Arn' --output text
```

**アクセスキーは1つも登録しない。** これが OIDC を使う理由である。

##### ⚠ モジュールを提供する側にジョブトークンを許可する（これを忘れると必ず 403 になる）

CI では `TF_TOKEN_gitlab_com` に `CI_JOB_TOKEN` を使う。ところが **GitLab のジョブトークンは、既定では自分のプロジェクトにしかアクセスできない。** 「どのプロジェクトの許可リストも、既定では自分自身しか含まない」のが GitLab の仕様である。

```
tf-catalog のジョブ  ──CI_JOB_TOKEN──▶  s3-bucket のパッケージレジストリ
                                        （別プロジェクト → 既定で拒否 = 403）
```

**設定するのは、モジュールを提供する `s3-bucket` プロジェクト側**である。失敗しているジョブの側ではない点に注意する。

1. **`<YOUR-GROUP>/s3-bucket`** プロジェクトを開く
1. 左サイドバー **設定 > CI/CD**
1. **ジョブトークンの権限（Job token permissions）** を展開
1. **CI/CD ジョブトークン許可リスト** の右の **追加** を選択
1. **グループまたはプロジェクト** を選び、`<YOUR-GROUP>/tf-catalog` を入力して **追加**

> **グループ単位で追加してもよい。** 許可リストにグループを入れると、そのグループとサブグループ配下の全プロジェクトが対象になり、**後から作ったプロジェクトも自動的に含まれる**。カタログ運用では利用者リポジトリが増え続けるので、**グループ単位で入れるほうが現実的**である。ただし範囲は広がるので、カタログ用のグループを分けておくのが望ましい。

> **長期トークンで回避してはいけない。** `CI_JOB_TOKEN` の代わりにデプロイトークンを CI 変数に置けば動くが、それは**この演習で OIDC を使って AWS の長期キーを排除した方針と逆行する**。ジョブトークンは実行中のジョブの寿命しか持たない使い捨ての資格情報であり、許可リストで範囲を絞るのが本来の設計である。

```bash
mkdir -p /c/dev/handson-terraform-beginner/tf-handson/tf-catalog && cd /c/dev/handson-terraform-beginner/tf-handson/tf-catalog
git init -b main
```

`main.tf`:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket       = "tfstate-handson-123456789012"
    key          = "catalog/dev/terraform.tfstate"
    region       = "ap-northeast-1"
    encrypt      = true
    use_lockfile = true
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

module "logs" {
  source  = "gitlab.com/acme-platform/s3-bucket/aws"
  version = "0.1.0"

  name        = "logs"
  environment = "dev"
  owner_team  = "team-alpha"
  cost_center = "CC-1234"
}

output "logs_bucket" {
  value = module.logs.bucket_id
}
```

#### 8-3. lock ファイルを Linux 向けにも作る（Windows 開発者の必須手順）

```bash
terraform init
```

`.terraform.lock.hcl` が生成される。**このファイルは必ず Git にコミットする**（プロバイダのバージョンとチェックサムを固定するため）。

中身を見ると、ハッシュが2種類記録されている。

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "6.63.0"
  constraints = ">= 5.0.0, ~> 6.0, < 7.0.0"
  hashes = [
    "h1:Pabctkahn1q6QN5LIAgQ3N7Co7a/4011OSE4K/3vKMU=",   ← 実行中のOS用（1つだけ）
    "zh:005d56736afd17d963998c405cee6f434dbc23a415109f9435ff1542879ae611",
    "zh:026ef126321a86ad7080b5d858e2527f96f5289678cbcd8856296e229c43339d",
    ...                                                    ← 全プラットフォーム分
  ]
}
```

| 種類 | 意味 |
|------|------|
| **`zh:`** | レジストリが公開している署名済みハッシュ。**プラットフォーム非依存** |
| **`h1:`** | 展開後のパッケージのハッシュ。**`init` を実行した OS の分しか記録されない** |

**Windows で `init` すると、`h1:` は Windows 用の1つしか入らない。** この状態で Linux の CI ランナーが `init` すると、`zh:` があるので検証自体は通り、**Linux 用の `h1:` を追記して先に進む**。CI のログにこう出る。

```
Terraform has made some changes to the provider dependency selections recorded
in the .terraform.lock.hcl file. Review those changes and commit them to your
version control system if they represent changes you intended to make.
```

> **「落ちないなら放置でいい」と思ってはいけない。** 実害は2つある。
>
> 1. **ロックファイルが収束しない。** CI のワークスペースは使い捨てなので、追記は毎回捨てられる。「バージョンとチェックサムを固定して再現性を担保する」という lock ファイル本来の目的を果たせていない。
> 2. **厳格化した瞬間に本当に落ちる。** プロバイダミラーや private registry を使う構成、あるいは CI を `terraform init -lockfile=readonly`（ロックファイルの変更を禁止するモード）にすると、これはハードエラーになる。カタログ運用では最終的に `-lockfile=readonly` を入れるべきなので、いま直しておく。

**対処**: 使うプラットフォーム分のハッシュをまとめて記録する。

```bash
terraform providers lock \
  -platform=windows_amd64 \
  -platform=linux_amd64 \
  -platform=darwin_arm64
```

```
- Obtained hashicorp/aws checksums for windows_amd64; All checksums for this platform were already tracked in the lock file
- Obtained hashicorp/aws checksums for linux_amd64; Additional checksums for this platform are now tracked in the lock file
- Obtained hashicorp/aws checksums for darwin_arm64; Additional checksums for this platform are now tracked in the lock file

Success! Terraform has updated the lock file.
```

`h1:` が3つに増えていることを確認してコミットする。

```bash
grep -c '"h1:' .terraform.lock.hcl     # 3
git add .terraform.lock.hcl
git commit -m "chore: 複数プラットフォームのプロバイダハッシュを記録"
```

> **これは Windows で開発して Linux の CI を回す全チームが踏む。** しかも**エラーにならず警告だけで進んでしまう**ため、気づかないまま何ヶ月も運用されがちである。CI ログに `Terraform has made some changes to the provider dependency selections` が出ていたら、それは「まだ直っていない」というサインだと覚えておく。

#### 8-4. パイプラインを書く

`.gitlab-ci.yml`:

```yaml
stages: [validate, plan, apply]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_IN_AUTOMATION: "true"
  AWS_DEFAULT_REGION: ap-northeast-1
  AWS_REGION: ap-northeast-1

default:
  image:
    name: hashicorp/terraform:1.14
    entrypoint: [""]
  id_tokens:
    AWS_ID_TOKEN:
      aud: https://gitlab.com
  before_script:
    # 設定漏れを、意味の分かるメッセージで早く落とす
    - |
      if [ -z "${AWS_ROLE_ARN}" ]; then
        echo "AWS_ROLE_ARN が空です。tf-catalog の 設定 > CI/CD > 変数 を確認してください。"
        echo "  - キー名が AWS_ROLE_ARN 丁度か（末尾スペースに注意）"
        echo "  - 「変数を保護する」がオンなら、このブランチが保護ブランチか"
        echo "  - 環境スコープが * （All）になっているか"
        exit 1
      fi
    # OIDC トークンをファイルに書き出すと、AWS プロバイダが自動で使ってくれる
    - echo "$AWS_ID_TOKEN" > /tmp/aws_web_identity_token
    - export AWS_WEB_IDENTITY_TOKEN_FILE=/tmp/aws_web_identity_token
    - export AWS_ROLE_SESSION_NAME="gitlab-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
    # Module Registry からモジュールを取得するための認証
    - export TF_TOKEN_gitlab_com="${CI_JOB_TOKEN}"
    - cd "${TF_ROOT}"
    - terraform init -input=false

fmt:
  stage: validate
  script:
    - terraform fmt -check -recursive -diff

validate:
  stage: validate
  script:
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -input=false -out=plan.tfplan
    - terraform show -no-color plan.tfplan > plan.txt
  artifacts:
    # plan には機微情報が含まれうる。Guest に見せない
    access: developer
    expire_in: 7 days
    paths:
      - ${TF_ROOT}/plan.tfplan
      - ${TF_ROOT}/plan.txt
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

apply:
  stage: apply
  script:
    - terraform apply -input=false plan.tfplan
  dependencies: [plan]
  environment:
    name: dev
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual        # 人間が最後のボタンを押す
      allow_failure: false
```

設計の要点を明示しておく。

| 決めたこと | 理由 |
|-----------|------|
| `plan` は MR とデフォルトブランチの両方で実行 | MR ではレビュー用、merge 後は apply の入力用 |
| `apply` は `plan.tfplan` を使う | **レビューされた計画と、実行される計画を一致させる**ため。`terraform apply` を裸で打つと再計算されてしまう |
| `apply` は `when: manual` | 完全自動化は次の段階。まず「人が押す」形で運用に慣れる |
| `artifacts:access: developer` | plan ファイルは暗号化されない。パスワードや証明書が含まれうる |
| `TF_TOKEN_gitlab_com=$CI_JOB_TOKEN` | Registry からモジュールを取るのに PAT を保存しなくてよい |
| `TF_IN_AUTOMATION=true` | 出力から対話向けのヒントが消えてログが読みやすくなる |

#### 8-5. 動かす

```bash
cat > .gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.*
plan.tfplan
plan.txt
EOF

git add -A
git commit -m "feat: カタログリポジトリの初版"
git remote add origin https://gitlab.com/acme-platform/tf-catalog.git
git push -u origin main
```

パイプラインが走ると、次の状態で**止まる**。

```
validate:  fmt ✅   validate ✅
plan:      plan ✅
apply:     apply ⚙  ← 手動ジョブ待ち

パイプライン全体のステータス: Blocked（ブロック済み）
```

> **`Blocked` は失敗ではない。** `when: manual` により「人がボタンを押すまで待つ」状態である。**これが演習8のゴールの姿**であって、エラーではない。

**ボタンを押す前に plan を読む。** ここを飛ばすと、この演習の意味が半分なくなる。**ジョブ** タブから `plan` ジョブを開き、ログ末尾か、右サイドバーの **ジョブアーティファクト** の `plan.txt` を見る。

| 見るところ | 期待値 |
|-----------|-------|
| 行頭の記号 | **`+` だけ**。`-` や `-/+` が無いこと（1.4 参照） |
| 作られるリソース | `module.logs.aws_s3_bucket.this` ほか計5つ |
| バケット名 | `acme-dev-logs-<ACCOUNT_ID>` |
| タグ | `OwnerTeam` `CostCenter` `CatalogItem` が入っていること |
| 最終行 | `Plan: 5 to add, 0 to change, 0 to destroy.` |

内容に納得したら `apply` の **▶** を押す。ジョブログに `Apply complete! Resources: 5 added, 0 changed, 0 destroyed.` が出れば成功。

手元から確認する。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3api get-bucket-tagging --bucket "acme-dev-logs-${ACCOUNT_ID}"
```

**アクセスキーを1つも保存せずに、CI が AWS にリソースを作った。** これが演習8の到達点である。

次に、MR での挙動を確認する。

```bash
git checkout -b feat/add-artifacts-bucket
```

`main.tf` に追記:

```hcl
module "artifacts" {
  source  = "gitlab.com/acme-platform/s3-bucket/aws"
  version = "0.1.0"

  name        = "artifacts"
  environment = "dev"
  owner_team  = "team-beta"
  cost_center = "CC-5678"

  retention_days = 14
}
```

```bash
git commit -am "feat: team-beta 用の artifacts バケットを追加"
git push -u origin feat/add-artifacts-bucket -o merge_request.create -o merge_request.target=main
```

MR のパイプラインで `plan` ジョブが走り、**ジョブのアーティファクト `plan.txt` にレビュー可能な差分**が残る。これが「MR で差分をレビューする」という運用の最小形である。

#### 8-6. 【参考】OpenTofu コンポーネントに乗り換える場合

同じことを GitLab 公式のコンポーネントでやると、`.gitlab-ci.yml` がこれだけになる。

```yaml
include:
  - component: $CI_SERVER_FQDN/components/opentofu/validate-plan-apply@4.9.0
    inputs:
      opentofu_version: "1.10.0"
      state_name: catalog-dev

stages: [validate, build, deploy]
```

| 得られるもの | 自前ジョブとの差 |
|------------|----------------|
| **MR ウィジェットへの plan サマリ表示** | 自前だと artifact を開く必要がある |
| GitLab 管理 state（HTTP backend）の自動設定 | `backend "http" {}` と書くだけでよくなる |
| ドリフト検知・OPA によるポリシー適用 | 自前だと別途実装 |
| 承認済み plan の再利用 | merge 後に plan を作り直さない |

**ただし CLI は `tofu` になる。** 社内標準を Terraform（BUSL）にするか OpenTofu（MPL）にするかは、ライセンス方針の話であって技術の話ではない。**このハンズオンで書いた `.tf` は、どちらでもそのまま動く。**

> **この演習で学んだこと**
> - **OIDC で AWS の長期キーを完全に排除できる。** `aud` は3箇所を一致させる。
> - `apply` は必ず**保存した plan ファイル**に対して実行する。
> - **`.terraform.lock.hcl` は複数プラットフォーム分を記録してコミットする**（Windows→Linux CI の必須手順）。
> - plan アーティファクトは機密扱い（`access: developer`）。

---

### 演習9: セルフサービス化 — 利用者は JSON を1枚置くだけ

**目的**: 演習8では、バケットを増やすのに `main.tf` を編集していた。これでは利用者に Terraform を書かせることになる。**申請ファイルを置くだけ**の形に変える。

#### 9-1. 申請ファイルの置き場を作る

```bash
cd /c/dev/handson-terraform-beginner/tf-handson/tf-catalog
git checkout main && git pull
mkdir -p requests/team-alpha requests/team-beta
```

`requests/team-alpha/logs.json`:

```json
{
  "name": "logs",
  "environment": "dev",
  "owner_team": "team-alpha",
  "cost_center": "CC-1234",
  "retention_days": 30
}
```

`requests/team-beta/artifacts.json`:

```json
{
  "name": "artifacts",
  "environment": "dev",
  "owner_team": "team-beta",
  "cost_center": "CC-5678",
  "retention_days": 14,
  "versioning_enabled": false
}
```

#### 9-2. 申請ファイルを読み込んで展開する

`main.tf` の `module` ブロックを、次のもので**丸ごと置き換える**。

```hcl
locals {
  # requests/ 配下の全 JSON を列挙する
  request_files = fileset("${path.module}/requests", "**/*.json")

  # ファイルパスを一意なキーにして map にする
  #   requests/team-alpha/logs.json  ->  "team-alpha/logs"
  requests = {
    for f in local.request_files :
    trimsuffix(f, ".json") => jsondecode(file("${path.module}/requests/${f}"))
  }
}

module "bucket" {
  # source / version にはリテラルしか書けない（演習7-3）
  source  = "gitlab.com/acme-platform/s3-bucket/aws"
  version = "0.1.0"

  for_each = local.requests

  name        = each.value.name
  environment = each.value.environment
  owner_team  = each.value.owner_team
  cost_center = each.value.cost_center

  # 省略可能な項目は try() で既定値にフォールバックする
  retention_days     = try(each.value.retention_days, 90)
  versioning_enabled = try(each.value.versioning_enabled, true)
  additional_tags    = try(each.value.additional_tags, {})
}

output "buckets" {
  description = "払い出されたバケットの一覧"
  value = {
    for k, m in module.bucket : k => m.bucket_id
  }
}
```

```bash
terraform init
terraform plan
```

```
Terraform will perform the following actions:

  # module.bucket["team-beta/artifacts"].aws_s3_bucket.this will be created
  ...
Plan: 5 to add, 0 to change, 0 to destroy.
```

**申請 JSON を1枚増やすだけで、モジュール呼び出しが1セット増える。** 利用者が触るのは `requests/` の中だけになった。

> **`for_each` のキー設計が最重要**: キーは `module.bucket["team-alpha/logs"]` のように **state 上のアドレス**になる。
> **ファイル名を変えるとキーが変わり、Terraform は「消して作り直す」と判断する。**
> 名前を変えたいときは `moved` ブロックで移動を宣言する。
> ```hcl
> moved {
>   from = module.bucket["team-alpha/logs"]
>   to   = module.bucket["team-alpha/app-logs"]
> }
> ```
> `count`（インデックスがキー）ではなく `for_each`（文字列がキー）を使うべき理由がここにある。**リストの途中に1件挿入すると、`count` では以降が全部ずれて作り直される。**

#### 9-3. 権限の境界を CODEOWNERS で引く

`CODEOWNERS`:

```
# プラットフォームチームの承認が必須な領域
/.gitlab-ci.yml       @acme-platform/platform
/main.tf              @acme-platform/platform
/CODEOWNERS           @acme-platform/platform
/.terraform.lock.hcl  @acme-platform/platform

# 申請ファイルは各チームが自分の領域を持つ
/requests/team-alpha/ @acme-platform/team-alpha
/requests/team-beta/  @acme-platform/team-beta
```

GitLab の **設定 > マージリクエスト** で「コードオーナーの承認を必須にする」を有効にする（Premium 以上）。Free プランの場合は、**保護ブランチと「承認者数 1 以上」の設定**で近い運用ができる。

> **ここが設計の分かれ目**: 「利用者の MR を無審査で通してよいか」。
> - **通してよい**と言えるのは、**モジュール側のガードレールが十分**な場合だけ（演習5でやったこと）。
> - **審査が要る**なら、それは「モジュールで表現しきれていない制約がある」というサインである。その制約を `validation` に落とせないか、まず考えるべき。
> - 現実解は多くの場合その中間で、**`dev` は無審査、`prd` はプラットフォーム承認必須**とする。これは CODEOWNERS では表現できないので、`requests/prd/` のようにディレクトリで環境を分けると実装しやすい。

#### 9-4. 利用者の体験を再現する

「利用者になったつもりで」申請してみる。

```bash
git checkout -b request/team-gamma-reports

mkdir -p requests/team-gamma
cat > requests/team-gamma/reports.json <<'EOF'
{
  "name": "reports",
  "environment": "dev",
  "owner_team": "team-gamma",
  "cost_center": "CC-9999",
  "retention_days": 365
}
EOF

git add requests/team-gamma/reports.json
git commit -m "request: team-gamma 用の reports バケットを申請"
git push -u origin request/team-gamma-reports -o merge_request.create -o merge_request.target=main
```

MR のパイプラインで `plan` が走り、`plan.txt` に次が出る。

```
  # module.bucket["team-gamma/reports"].aws_s3_bucket.this will be created
  + resource "aws_s3_bucket" "this" {
      + bucket        = "acme-dev-reports-123456789012"
      + force_destroy = false
      + tags          = {
          + "CatalogItem" = "s3-bucket"
          + "CostCenter"  = "CC-9999"
          + "Environment" = "dev"
          + "ManagedBy"   = "terraform"
          + "OwnerTeam"   = "team-gamma"
        }
    }

Plan: 5 to add, 0 to change, 0 to destroy.
```

**レビュアーがやることは「この差分が申請どおりか」を見るだけ。** 命名も暗号化もタグも、機械が保証している。

#### 9-5. 不正な申請が弾かれることを確認する

```bash
cat > requests/team-gamma/bad.json <<'EOF'
{
  "name": "Reports_2024",
  "environment": "production",
  "owner_team": "gamma",
  "cost_center": "9999"
}
EOF

terraform plan
```

```
Error: Invalid value for variable

  on main.tf line 34, in module "bucket":
  34:   name        = each.value.name
    ├────────────────
    │ var.name is "Reports_2024"

name は小文字英数字とハイフンのみ、3〜21文字、先頭は英字にしてください。

This was checked by the validation rule at
.terraform/modules/bucket/variables.tf:11,3-13.

Error: Invalid value for variable
  ... environment は dev / stg / prd のいずれかです。

Error: Invalid value for variable
  ... owner_team は team-xxx の形式で指定してください。

Error: Invalid value for variable
  ... cost_center は CC-1234 の形式で指定してください。
```

**4件の規約違反が、AWS のリソースに一切触れることなく、MR のパイプラインで検出された。** レビュアーは何も指摘しなくてよい。

> **細かいが役に立つ挙動**: 変数の `validation` は **`plan` の最初、AWS API を呼ぶ前に評価される**。上のエラーは AWS の認証情報が無い環境でも同じように出る（認証エラーと並んで表示される）。つまり **CI の `plan` ジョブが AWS に繋がらなくても、規約違反だけは検出できる**。

```bash
rm requests/team-gamma/bad.json
```

#### 9-6. 現時点の限界を認識しておく

この構成には、**本番運用の前に必ず向き合うべき弱点**がある。隠さずに書いておく。

| 弱点 | 何が起きるか | 次の一手 |
|------|------------|---------|
| **全チームが1つの state に乗っている** | team-alpha の apply 失敗が team-beta をブロックする。state が壊れると全滅 | チーム別／環境別に state を分割する（ディレクトリ + 子パイプライン） |
| **`plan` の実行時間が線形に伸びる** | 申請が200件を超えると `refresh` だけで数分 | `-target` の常用は避け、state 分割で解く |
| **削除が「JSON を消す」で起きる** | 誤ってファイルを消すと本番バケットが destroy される | `prevent_destroy`、`prd` の apply を別パイプラインにする |
| **モジュールのバージョンが1箇所** | 上げると全チームに一斉に影響する | 環境ごとにディレクトリを分け、段階的に上げる |
| **申請の重複・払い出し済みの可視化がない** | 「誰が何を持っているか」は state を見ないと分からない | `output` を dotenv 化して一覧を生成する／タグベースで棚卸しする |

> **正直なところ**: ここまでの構成は **50〜100 リソース規模の「小さな社内カタログ」なら十分に実用**である。それを超えると state 分割の設計が本題になる。**最初から分割設計をやろうとして着手できないより、この形で始めて限界が見えてから割るほうが速い。**

> **この演習で学んだこと**
> - `fileset` + `jsondecode` + `for_each` で「申請ファイル駆動」が作れる。
> - **`for_each` のキーは state 上のアドレス**。ファイル名の変更は作り直しを意味する。`moved` で回避する。
> - `try()` で省略可能な項目に既定値を与える。
> - ガードレールがモジュール側にあるからこそ、**MR のレビューが軽くなる**。
> - 1つの state に全部乗せる構成には明確な限界がある。

---

### 演習10: 片付け

**目的**: 課金とトークンを残さない。

```bash
# 1. カタログが作ったリソースを削除
cd /c/dev/handson-terraform-beginner/tf-handson/tf-catalog
terraform destroy
```

`aws_s3_bucket` にオブジェクトが入っていると destroy は失敗する。その場合は中身を空にする。

```bash
for b in $(terraform output -json buckets | jq -r '.[]'); do
  aws s3 rm "s3://$b" --recursive
done
terraform destroy -auto-approve
```

```bash
# 2. 残っている演習ディレクトリを確認して destroy
for d in /c/dev/handson-terraform-beginner/tf-handson/ex*; do
  if [ -f "$d/terraform.tfstate" ] || [ -f "$d/backend.tf" ]; then
    echo "=== $d"
    (cd "$d" && terraform destroy -auto-approve)
  fi
done
```

```bash
# 3. state バケットを削除（バージョニング有効なので全バージョンを消す必要がある）
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
STATE_BUCKET="tfstate-handson-${ACCOUNT_ID}"

aws s3api delete-objects --bucket "$STATE_BUCKET" \
  --delete "$(aws s3api list-object-versions --bucket "$STATE_BUCKET" \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)" 2>/dev/null

aws s3api delete-objects --bucket "$STATE_BUCKET" \
  --delete "$(aws s3api list-object-versions --bucket "$STATE_BUCKET" \
    --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json)" 2>/dev/null

aws s3api delete-bucket --bucket "$STATE_BUCKET"
```

```bash
# 4. IAM ロールと OIDC プロバイダを削除
aws iam detach-role-policy --role-name gitlab-tf-catalog \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam delete-role --role-name gitlab-tf-catalog

aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/gitlab.com"
```

```bash
# 5. 取り残しがないか確認
aws s3 ls | grep -E 'acme-|tfstate-handson|tf-handson' || echo "S3: clean"
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=ManagedBy,Values=terraform \
  --query 'ResourceTagMappingList[].ResourceARN' --output table
```

**6. GitLab 側**

- 演習7で発行した個人アクセストークン／デプロイトークンを**失効させる**
- 不要なら `s3-bucket` / `tf-catalog` プロジェクトを削除する

> **最後の確認**: `~/.terraformrc`（または `%APPDATA%/terraform.rc`）にトークンを書いた場合は消しておく。

---

## 4. 習得事項のまとめ

### 4.1 触れた要素の一覧

| 要素 | 役割 | 登場した演習 |
|------|------|------------|
| `terraform.tfstate` | 管理対象の台帳。**Terraform の本体** | 1, 4 |
| `terraform state list / show / rm` | state の中身の確認と操作 | 1 |
| `-refresh=false` | 現実の読み直しを省略（速いがドリフトを見逃す） | 1 |
| `-replace=` | 強制的に作り直す（旧 `taint`） | 1 |
| `default_tags` | プロバイダ単位の共通タグ | 2, 3 |
| `# forces replacement` | 作り直しの原因を示す plan 内のマーカー | 2 |
| `validation` | クラウドに触れる前のガードレール | 3, 5, 9 |
| 変数の優先順位（6段階） | `default` < 環境変数 < tfvars < auto.tfvars < `-var` | 3 |
| `sensitive` | 一覧表示だけを隠す（**名指しすれば出る**。state には平文で残る） | 3 |
| `backend "s3"` + `use_lockfile` | リモート state とネイティブロック | 4 |
| `terraform init -migrate-state` | backend の移行 | 4 |
| `terraform force-unlock` | 残留ロックの解除（要注意） | 4 |
| モジュール（`required_providers` のみ） | 再利用の単位＝カタログの商品 | 5 |
| `data "aws_caller_identity"` | 既存の情報を読む | 5 |
| Terraform Module Registry | バージョン付きの商品棚 | 6, 7 |
| `TF_TOKEN_<host>` | Registry 認証 | 7 |
| `id_tokens` + OIDC | **AWS 長期キーの排除** | 8 |
| `terraform plan -out` → `apply plan.tfplan` | レビューした計画をそのまま実行 | 8 |
| `terraform providers lock -platform=` | 複数 OS 向けのハッシュ記録 | 8 |
| `artifacts:access: developer` | plan の秘匿 | 8 |
| `fileset` + `jsondecode` + `for_each` | 申請ファイル駆動の払い出し | 9 |
| `moved` | リソースの改名を「移動」として宣言 | 9 |
| `try()` | 省略可能な項目のフォールバック | 9 |

### 4.2 Terraform の勘所（忘れやすい順）

1. **state を失う＝管理権を失う。** リソースは残る。だから backend にはバージョニングを付ける。
2. **`sensitive` はアクセス制御ではない。** 一覧では隠れるが `terraform output <名前>` や `-json` では出るし、state には平文で残る。守るのは state 側。
3. **モジュールに `provider` を書かない。** 書くと `for_each` が使えず、削除もできなくなる。
4. **`source` / `version` に変数は使えない。** モジュール解決は変数評価より前。
5. **`count` ではなく `for_each` を使う。** インデックスがキーだと、途中への挿入で全部ずれる。
6. **`-/+` と `# forces replacement` を plan で探す。** サマリの数字だけ見ない。
7. **`apply` は保存した plan ファイルに対して行う。** そうしないとレビューした内容と実行内容がズレる。
8. **バージョン制約は「モジュールは緩く、ルートは固く」。**
9. **`.terraform.lock.hcl` はコミットする。** Windows で開発するなら `-platform=linux_amd64` も記録する。
10. **`terraform force-unlock` は最後の手段。** 動いている apply を解除すると state が壊れる。

### 4.3 カタログ設計の勘所

| 原則 | 意味 |
|------|------|
| **選ばせる項目を最小にする** | 選択肢の数だけレビューの負荷とバグの余地が増える |
| **禁止は文書ではなくコードで表現する** | Wiki の規約は守られない。`validation` は回避できない |
| **必須タグを「必須変数」にする** | 「持ち主のいないリソース」を構造的に作れなくする |
| **`prd` だけ扱いを変える** | 環境で危険度が違う。`validation` の変数間参照（1.9+）で表現できる |
| **商品説明書（README）を書く** | Module Registry のカタログ画面に出る。これが利用者との唯一の接点 |
| **バージョンは上書きしない** | 発行済みモジュールは編集不可。semver を守る意味がここにある |
| **小さく始めて、限界が見えてから割る** | state 分割は最初から完璧に設計できない |

### 4.4 トラブルシューティング

| 症状・エラー | 原因 | 対処 |
|------------|------|------|
| `Unsupported block type: versioning` | AWS プロバイダ v4 以降で S3 の付帯設定が別リソースに分離された | `aws_s3_bucket_versioning` 等の別リソースを使う |
| CI のログに `Terraform has made some changes to the provider dependency selections recorded in the .terraform.lock.hcl file` | lock ファイルに CI（Linux）用の `h1:` ハッシュが無く、毎回追記されては捨てられている。**エラーにならないので見落とされやすい** | `terraform providers lock -platform=windows_amd64 -platform=linux_amd64` を実行してコミット |
| `Provider ... does not have a package available for your current platform` / `doesn't match any of the checksums previously recorded` | 上と同じ原因だが、プロバイダミラー・private registry・`-lockfile=readonly` を使っている場合はハードエラーになる | 同上 |
| `Error acquiring the state lock` | 別プロセスが実行中、または異常終了でロックが残った | 実行中でないことを確認し `terraform force-unlock <ID>` |
| `BucketAlreadyOwnedByYou` | state を失った状態で apply した | state を復旧するか `terraform import` で取り込む |
| `Error: Variables not allowed`（`source` 行） | `source` / `version` に変数を使った | リテラル文字列にする。分岐が必要ならディレクトリを分ける |
| `A module with the same name already exists in the namespace.` | 同一トップレベルグループ内でモジュール名が重複 | 名前を変えるか、グループ設定で重複を許可する |
| Module Registry から **401** Unauthorized | `TF_TOKEN_gitlab_com` 未設定、変数名のドット変換ミス、別シェルで `export` した | ホスト名のドットを `_` に。`echo "set=\"` で同じシェルに生きているか確認 |
| **CI の `init`** で Module Registry から **403** Forbidden | `CI_JOB_TOKEN` は既定で自分のプロジェクトにしかアクセスできない | **モジュール提供側**（`s3-bucket`）の **設定 > CI/CD > ジョブトークンの権限 > CI/CD ジョブトークン許可リスト**に、利用側のプロジェクトかグループを追加する |
| **ローカルの `init`** で Module Registry から **403** Forbidden | トークンは届いている。**① 名前空間（グループパス）が違う**（GitLab は存在しないパスにも 403 を返す）、または **② スコープ不足**（`read_repository` や `read_registry` を選んでいる） | **まず `source` のグループ名をレジストリ画面からコピペで貼り直す。** それでも駄目ならスコープを PAT は `read_api`、デプロイトークンは `read_package_registry` に |
| `does not have a package for the given version` | タグに `v` を付けた（`v0.1.0`） | `v` 無しの semver でタグを打ち直す |
| `Error: role ARN is not set`（CI の `init`） | `AWS_WEB_IDENTITY_TOKEN_FILE` はあるが `AWS_ROLE_ARN` が空 | CI/CD 変数 `AWS_ROLE_ARN` を確認。**キー名／「変数を保護する」／環境スコープ `*`** の3点を疑う |
| `InvalidIdentityToken`（CI の AWS 認証） | `aud` が ID プロバイダ／信頼ポリシー／`id_tokens` で不一致 | 3箇所を同じ値にする |
| `Error: Provider configuration not present`（モジュール削除時） | モジュール内に `provider` ブロックを書いていた | モジュールから `provider` を消す。既存分は一度 provider を残して destroy |
| push 時に `LF will be replaced by CRLF the next time Git touches it` | **警告であって異常ではない。** コミットには LF で入っている | 気にしなくてよい。作業コピーも LF に揃えたければリポジトリ直下に `.gitattributes`（`*.tf text eol=lf`） |
| `fatal: The current branch <name> has no upstream branch` | 新規ブランチの初回 push でリモートとブランチを指定していない | `git push -u origin <ブランチ名>` を付ける。`-o merge_request.create` はその後ろに続ける |
| `plan` は通るが `apply` で権限エラー | IAM ポリシーが読み取りのみ | `plan` は Read、`apply` は Write が要る。CI ロールの権限を見直す |
| `Error: Output "X" not found` | `output` ブロックを追記しただけで `apply` していない | 出力は state に保存される値。**リソースの変更が0件でも `terraform apply` が必要** |
| `state snapshot was created by Terraform vX, which is newer than current vY` | 誰かが新しい Terraform で apply し、state のバージョンが上がった | **state は下げられない。** 全員と CI を同じバージョンに合わせる。`required_version = "~> 1.14"` のように上限も切って再発を防ぐ |

### 4.5 実務への持ち込み方（段階案）

いきなり演習9の形を目指さないこと。次の順で入れると、各段階で単独の価値が出る。

| 段階 | やること | 得られる価値 | 期間の目安 |
|:---:|---------|------------|:---:|
| **1** | 既存リソースを1種類だけモジュール化し、GitLab に publish する | 「社内標準の作り方」が形になる | 1〜2週 |
| **2** | 自分のチームの1環境を、そのモジュールで作り直す（`import` で取り込む） | ドッグフーディング。使えない設計が露見する | 2〜4週 |
| **3** | state を S3（または GitLab 管理 state）に移し、CI で plan だけ回す | 変更が MR でレビューできるようになる | 1〜2週 |
| **4** | `apply` を手動ジョブとして CI に載せる。OIDC で長期キーを排除 | ローカル apply をやめられる | 2週 |
| **5** | 申請ファイル駆動にして、他チームに開く | ここで初めて「カタログ」になる | 1〜2ヶ月 |

> **段階2を飛ばさないこと。** 自分たちが使っていないモジュールを他チームに配ると、必ず「実際には使えない」設計になっている。

---

## 5. 今後の学習ロードマップ

### 優先度順の次のステップ

| ★ | テーマ | なぜ次にやるか | 入口 |
|:--:|--------|--------------|------|
| **★1** | **静的チェックを CI に足す** | `validation` で書けないルール（コスト、命名の全社規約）を機械化する | `tflint`, `trivy config`, `checkov` を `validate` ステージに追加 |
| **★1** | **`terraform test` でモジュールを検証する** | 「validation が本当に効くか」をテストできる。カタログの商品には回帰テストが要る | `tests/*.tftest.hcl` を書き、publish 前の CI で実行 |
| **★2** | **state の分割戦略** | 演習9の最大の弱点。チーム別・環境別・ライフサイクル別のどれで割るか | ディレクトリ分割 + 子パイプライン、`terraform_remote_state` は使いすぎない |
| **★2** | **ポリシー as code** | 「タグの有無」を超えた組織ポリシー（許可リージョン、インスタンスサイズ上限） | OPA / Conftest を plan の JSON に対して実行。OpenTofu コンポーネントには組込みの対応がある |
| **★2** | **ドリフト検知の定期実行** | コンソールでの手作業を発見する。カタログの信頼性の根拠になる | スケジュールパイプラインで `plan -detailed-exitcode` |
| **★3** | **既存リソースの取り込み** | 「棚卸し型」（1.6 の C）への発展。手作業で作った既存資産をカタログに合流させる | `import` ブロック、`terraform plan -generate-config-out=`、**Terraform 1.14 の `terraform query` / `.tfquery.hcl`** |
| **★3** | **商品の追加** | S3 だけでは「カタログ」と呼びにくい | VPC、ECR、IAM ロール、RDS の順に増やす。**依存のある商品ほど設計が難しい** |
| **★3** | **GitLab CI/CD カタログとの統合** | パイプライン自体も再利用可能な部品にする | CI/CD コンポーネントを自作して公開する |
| **★4** | **AWS Service Catalog / Proton との比較** | 「Terraform で自作」が本当に正解かを検証する。AWS 側にも同じ課題の解がある | AWS Service Catalog の Terraform サポート、Backstage 等の開発者ポータル |

### 参考リンク

**Terraform 本体**

- [Terraform 公式ドキュメント](https://developer.hashicorp.com/terraform/docs)
- [S3 バックエンド（`use_lockfile` を含む）](https://developer.hashicorp.com/terraform/language/backend/s3)
- [モジュールの作り方](https://developer.hashicorp.com/terraform/language/modules/develop)
- [`terraform test`](https://developer.hashicorp.com/terraform/language/tests)
- [Terraform v1.14.0 リリースノート（List Resources / Actions）](https://discuss.hashicorp.com/t/terraform-v1-14-0-released/76808)

**AWS プロバイダ**

- [AWS Provider ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [`aws_s3_bucket`（v4 以降の分離について）](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)

**GitLab**

- [Terraform Module Registry](https://docs.gitlab.com/user/packages/terraform_module_registry/)
- [GitLab 管理の Terraform/OpenTofu state](https://docs.gitlab.com/user/infrastructure/iac/terraform_state.md)
- [OpenTofu CI/CD コンポーネント](https://gitlab.com/components/opentofu)
- [AWS で OpenID Connect を設定する](https://docs.gitlab.com/ci/cloud_services/aws/)

**OpenTofu**

- [OpenTofu 公式](https://opentofu.org/docs/)
- [Module Registry Protocol](https://opentofu.org/docs/internals/module-registry-protocol/)

---

> 図を編集したい場合は、`diagrams/` 内の PNG を draw.io（[app.diagrams.net](https://app.diagrams.net)）または draw.io デスクトップアプリで開くと、埋め込まれた図データをそのまま編集できます。
