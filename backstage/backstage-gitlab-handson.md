# Backstage ハンズオン — GitLab をバックエンドに IDP(社内開発者ポータル)を立ち上げる

> 対象レベル: 中級(Git / ターミナル操作 / YAML の基本が分かる方)
> 所要時間: 約 2.5 〜 3 時間
> 使用ターミナル: **Git Bash**(コマンド例はすべて Git Bash 前提)

---

## 1. 勉強対象の概要

### 1.1 IDP(Internal Developer Platform)とは

IDP は「開発者が自分たちでサービスの作成・管理・運用をセルフサービスでできるようにする社内基盤」です。プラットフォームエンジニアリングの中核となる考え方で、次の課題を解決します。

| よくある課題 | IDP による解決 |
|---|---|
| どのチームがどのサービスを持っているか分からない | **サービスカタログ**で全サービスと所有者を一覧化 |
| 新規サービスの立ち上げ方がチームごとにバラバラ | **テンプレート**で標準構成をワンクリック生成(ゴールデンパス) |
| ドキュメントが散在して見つからない | ドキュメントをコードと一緒に管理し、ポータルに集約 |
| 認知負荷が高い(K8s / CI / クラウドを全員が覚える) | プラットフォームチームが「舗装された道」を提供 |

### 1.2 Backstage とは

[Backstage](https://backstage.io/) は Spotify が開発し CNCF に寄贈した、**IDP(開発者ポータル)を構築するためのオープンソースフレームワーク**です。Node.js(React + Express)製で、プラグインで機能を拡張します。

押さえるべき中心概念は次の 6 つです。

| 概念 | 説明 |
|---|---|
| **Software Catalog** | 組織内のサービス・API・リソースを一元管理する台帳。本ハンズオンの主役 |
| **エンティティ / catalog-info.yaml** | カタログの 1 項目(Component, API, System など)。各リポジトリに置く `catalog-info.yaml` で定義する |
| **Scaffolder (Software Templates)** | テンプレートから新規プロジェクトを自動生成する仕組み。「新規サービス作成のセルフサービス化」を実現 |
| **TechDocs** | Markdown(MkDocs)ベースのドキュメントをポータル内に表示する仕組み(docs-like-code) |
| **Integrations** | GitLab / GitHub などの外部サービスとの接続設定。トークンを設定して API 連携する |
| **プラグイン** | 機能拡張の単位。カタログも Scaffolder も実はプラグイン。GitLab CI パイプライン表示などのコミュニティプラグインも多数 |

### 1.3 全体アーキテクチャ(このハンズオンで作るもの)

ローカル PC で Backstage を起動し、GitLab(gitlab.com)をソース管理バックエンドとして連携させます。

![BackstageによるIDPの全体像](images/idp-overview.png)

- Backstage 本体はローカルの Node.js アプリ(フロントエンド + バックエンド)として動作
- GitLab とは **Personal Access Token(PAT)** を使った REST API 連携
- リポジトリ内の `catalog-info.yaml` をカタログに取り込み、逆にテンプレートから GitLab へリポジトリを自動作成する

---

## 2. ハンズオンの概要

### 2.1 想定環境

| 項目 | 内容 |
|---|---|
| OS | Windows 11(Git Bash を使用)/ macOS・Linux でもほぼ同じ手順 |
| Node.js | **22.x(Active LTS)** ※ Backstage は Node.js 22 / 24 をサポート |
| パッケージ管理 | yarn(corepack 経由で導入) |
| Git | インストール済みであること |
| GitLab | **gitlab.com の無料アカウント**(Self-managed でも手順はほぼ同じ) |
| メモリ | 8GB 以上推奨(初回ビルドが重いため) |
| Docker | **不要**(ローカル起動は SQLite を使用) |

### 2.2 ゴールイメージ

ハンズオン終了時には、次の状態ができあがっています。

1. ローカルで Backstage ポータルが起動している(`http://localhost:3000`)
2. GitLab 上のリポジトリが **Software Catalog に登録**されている(手動登録 + 自動ディスカバリーの両方)
3. Backstage の画面から **Software Template を実行すると、GitLab に新規リポジトリが自動作成され、カタログにも自動登録される**(セルフサービスの体験)

### 2.3 学べることの全体像

| 演習 | やること | 学べること |
|---|---|---|
| 事前準備 | Node.js / GitLab アカウント / PAT 準備 | Backstage 動作要件、GitLab トークンのスコープ設計 |
| 演習 1 | `create-app` で Backstage を起動 | Backstage の構成(app / backend)、ローカル起動の仕組み |
| 演習 2 | `integrations.gitlab` を設定 | Integrations の役割、設定ファイルのレイヤ(app-config.local.yaml) |
| 演習 3 | catalog-info.yaml を書いて手動登録 | エンティティモデル、カタログ取り込みの流れ |
| 演習 4 | GitLab Discovery で自動取り込み | Entity Provider、宣言的なカタログ運用 |
| 演習 5 | Software Template で新規リポジトリ作成 | Scaffolder、`publish:gitlab` アクション、ゴールデンパス |
| 演習 6(任意) | エンティティ間の関係を定義 | System / dependsOn とリレーショングラフ |

### 2.4 演習全体の流れ

![ハンズオン全体の流れ](images/handson-flow.png)

---

## 3. ハンズオンの手順

### 事前準備

#### (1) Node.js 22 の確認

Git Bash で確認します。

```bash
node -v    # v22.x.x であること(v24 でも可)
git --version
```

Node.js が古い場合は [nodejs.org](https://nodejs.org/) から 22 LTS をインストールするか、[nvm-windows](https://github.com/coreybutler/nvm-windows) や [Volta](https://volta.sh/) でバージョン切り替えできるようにしておくと便利です。

#### (2) yarn の準備

Backstage は yarn を使います。corepack 経由で有効化します(管理者権限の Git Bash が必要な場合があります)。

```bash
corepack enable
yarn --version
```

#### (3) GitLab アカウントとグループの確認

1. [gitlab.com](https://gitlab.com/) にサインイン(無料アカウントで OK)
2. 本ハンズオンでは既存グループ **`handson-idp-catalog`** を使用します(左上メニュー → **Groups** で存在を確認)
   - ※ 演習 4 の自動ディスカバリーは「グループ配下のプロジェクト」を対象にするため、演習で作るプロジェクトはすべてこのグループ配下に作成します
   - 別のグループ名を使う場合は、以降の `handson-idp-catalog` を読み替えてください(サブグループの場合は `parent/child` のフルパス)

#### (4) Personal Access Token(PAT)の発行

1. GitLab 右上のアバター → **Edit profile → Access tokens → Add new token**
2. 以下で作成します

| 項目 | 値 |
|---|---|
| Token name | `backstage-handson` |
| Expiration date | 適当な近い日付(ハンズオン後に失効させる) |
| Scopes | **`api`, `read_repository`, `write_repository`** の 3 つにチェック |

> 💡 細粒度(fine-grained)トークンの作成画面が表示された場合は、**「レガシートークン」を選択**してください(Backstage は従来型スコープを前提としています)。また `read_repository` と `read_registry` は名前が似ているので選び間違いに注意。

正しくスコープを選択した状態の例:

![GitLab レガシートークンのスコープ設定例](images/gitlab-pat-scopes.png)

3. 表示されたトークン(`glpat-...`)を控えます。**この画面を閉じると二度と表示されません。**

> ⚠️ トークンは Git にコミットしない・チャットに貼らないこと。本ハンズオンでは環境変数で渡します。

✅ **確認ポイント**: `node -v` が v22 系、yarn が動作、GitLab にグループが存在し、PAT を控えてある。

---

### 演習 1: Backstage アプリを作成して起動する

**目的**: Backstage アプリの雛形を作成し、ローカルでポータルが動く状態にする。Backstage の基本構成(app / backend)を把握する。

#### 手順

1. 作業ディレクトリでアプリを作成します(数分かかります)。

```bash
cd /c/dev/handson-backstage
npx @backstage/create-app@latest
```

2. アプリ名を聞かれるので `my-idp` と入力します。依存関係のインストールが自動で走ります。

3. 起動します(初回は TypeScript のビルドで数分かかります)。

```bash
cd my-idp
yarn start
```

4. 自動でブラウザが開きます(開かなければ `http://localhost:3000` へ)。サインイン画面が出たら **Enter(guest)** で入ります。

#### 生成されたものを眺める

```text
my-idp/
├── app-config.yaml           # メイン設定ファイル
├── app-config.production.yaml # 本番用の上書き設定
├── catalog-info.yaml         # Backstage 自身のカタログ定義
├── examples/                 # サンプルのエンティティ・テンプレート
└── packages/
    ├── app/                  # フロントエンド(React)
    └── backend/              # バックエンド(Node.js)← 演習 4 で触る
```

✅ **確認ポイント**
- `http://localhost:3000` でポータルが表示される
- 左メニューの **Catalog** にサンプルエンティティ(`example-website` など)が表示される
- **Create...** にサンプルテンプレートが表示される

> 💡 **ここで学んだこと**: Backstage は「フロントエンド(app)+ バックエンド(backend)」のモノレポ構成。ローカルでは SQLite を内蔵 DB として使うため、Docker なしで起動できる。

---

### 演習 2: GitLab インテグレーションを設定する

**目的**: Backstage が GitLab API を呼べるようにする。設定ファイルのレイヤ構造(app-config.yaml と app-config.local.yaml)を理解する。

#### 手順

1. `my-idp/app-config.local.yaml` を開きます(**存在しない場合は `my-idp` 直下に新規作成**。このファイルは `.gitignore` 済みなのでトークン設定に適していますが、今回は環境変数参照にしてさらに安全にします)。次を記述します。

```yaml
integrations:
  gitlab:
    - host: gitlab.com
      token: ${GITLAB_TOKEN}
```

> Self-managed GitLab の場合は `host` を自社ドメインにし、必要に応じて `apiBaseUrl: https://<host>/api/v4` を追記します。

2. `yarn start` を実行しているターミナルを `Ctrl + C` で止め、**同じターミナルで**環境変数を設定してから再起動します。

```bash
export GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx   # 事前準備で控えたトークン
yarn start
```

> 💡 毎回 export するのが面倒なら `~/.bashrc` に追記しても構いません(共有 PC では非推奨)。

✅ **確認ポイント**: 起動ログにエラーが出ないこと。この時点では画面上の変化はありません(効果は演習 3 以降で現れます)。

> 💡 **ここで学んだこと**: `integrations` は「Backstage と外部サービスの接続口」。カタログ取り込みも Scaffolder も、ここで設定したトークンを共通で使う。設定は `app-config.yaml`(共通)← `app-config.local.yaml`(ローカル上書き)の順にマージされる。

---

### 演習 3: GitLab のリポジトリをカタログに手動登録する

**目的**: エンティティ定義(`catalog-info.yaml`)を自分で書き、URL 指定でカタログに取り込む。カタログの取り込みフローを理解する。

カタログ取り込みの内部の流れは次のとおりです。

![Software Catalogがエンティティを取り込む流れ](images/catalog-flow.png)

#### 手順

1. GitLab で、`handson-idp-catalog` グループ配下に**新規プロジェクト**を作成します。
   - **New project → Create blank project**
   - Project name: `sample-service`
   - 「Initialize repository with a README」にチェック

2. GitLab の Web IDE またはファイル追加機能(**+ → New file**)で、リポジトリ直下に `catalog-info.yaml` を作成しコミットします。

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: sample-service
  description: ハンズオン用のサンプルサービス
  annotations:
    gitlab.com/project-slug: handson-idp-catalog/sample-service
  tags:
    - handson
spec:
  type: service
  lifecycle: experimental
  owner: guests
```

主なフィールドの意味:

| フィールド | 意味 |
|---|---|
| `kind: Component` | エンティティの種類(他に `API`, `System`, `Resource`, `Group`, `User` など) |
| `metadata.name` | カタログ上の一意な名前 |
| `metadata.annotations` | プラグインが使う付加情報。`gitlab.com/project-slug` は GitLab 系プラグインの紐付けキー |
| `spec.type` | Component の分類(`service` / `website` / `library` など) |
| `spec.lifecycle` | ライフサイクル(`experimental` / `production` / `deprecated`) |
| `spec.owner` | 所有者(カタログ上の Group/User 名。今回は雛形に含まれる `guests` グループ) |

3. Backstage の画面で登録します。
   - 左メニュー **Create...** → 右上の **Register Existing Component**
   - URL に catalog-info.yaml のファイル URL を入力します

```text
https://gitlab.com/handson-idp-catalog/sample-service/-/blob/main/catalog-info.yaml
```

   - **Analyze** → 内容を確認して **Import**

✅ **確認ポイント**
- **Catalog** 画面に `sample-service` が表示される(下図はインポート成功後の状態)

![カタログにsample-serviceが登録された状態](images/catalog-sample-service.png)

- クリックすると Overview に説明・タグ・オーナーが表示される
- Private プロジェクトなのに読めている = 演習 2 のトークンが効いている証拠

> 💡 **ここで学んだこと**: カタログの実体は「各リポジトリに置かれた catalog-info.yaml」。カタログ側にデータを直接入力するのではなく、**ソースコード側に宣言を置いて取り込む**のが Backstage 流(single source of truth)。

---

### 演習 4: GitLab Discovery で自動ディスカバリーする

**目的**: リポジトリを 1 つずつ手動登録するのではなく、**グループ配下のリポジトリを定期スキャンして自動でカタログに取り込む**仕組み(Entity Provider)を導入する。

#### 手順

1. `Ctrl + C` で停止し、GitLab カタログモジュールをバックエンドに追加します。

```bash
yarn --cwd packages/backend add @backstage/plugin-catalog-backend-module-gitlab
```

2. `packages/backend/src/index.ts` を開き、他の `backend.add(...)` が並んでいる箇所(catalog 関連の近く)に 1 行追加します。

```typescript
backend.add(import('@backstage/plugin-catalog-backend-module-gitlab'));
```

3. `app-config.local.yaml` に Discovery プロバイダーの設定を追記します(`integrations` と同じファイルで OK)。

```yaml
catalog:
  providers:
    gitlab:
      myGitlabProvider:                    # 任意の ID
        host: gitlab.com                   # integrations.gitlab の host と一致させる
        group: handson-idp-catalog                # スキャン対象のグループパス
        branch: main                       # catalog-info.yaml を探すブランチ
        entityFilename: catalog-info.yaml  # 既定値なので省略可
        schedule:
          frequency: { minutes: 3 }        # ハンズオン用に短め(実運用は 30 分程度)
          timeout: { minutes: 3 }
```

4. 再起動します。

```bash
export GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
yarn start
```

5. 自動取り込みを体験するため、GitLab に **2 つ目のプロジェクト** `sample-service-2` を作成し、演習 3 と同様の `catalog-info.yaml` を置きます(`metadata.name` と `project-slug` は `sample-service-2` に変えること)。

6. **何も登録操作をせずに** 3 分ほど待ちます(またはバックエンド再起動で即時スキャン)。

✅ **確認ポイント**
- Catalog 画面に `sample-service-2` が**自動で**現れる
- バックエンドのログに `GitlabDiscoveryEntityProvider` 系の処理ログが出ている

> 💡 **ここで学んだこと**: Entity Provider は「外部システムを定期スキャンしてエンティティを供給する」コンポーネント。これにより「リポジトリに catalog-info.yaml を置けば勝手にカタログに載る」という宣言的な運用が実現でき、IDP のスケールに不可欠。

---

### 演習 5: Software Template で新規リポジトリ作成をセルフサービス化する

**目的**: Backstage の画面から入力フォームに答えるだけで、**GitLab に標準構成の新規リポジトリが作成され、カタログにも自動登録される**ゴールデンパスを構築する。

テンプレート実行の流れは次のとおりです。

![Software Template実行の流れ](images/scaffolder-flow.png)

#### 手順

1. `Ctrl + C` で停止し、**Scaffolder 用の GitLab アクションモジュールを追加**します。
   ※ 演習 4 で入れたのはカタログ(Discovery)用モジュールで、`publish:gitlab` などの Scaffolder アクションは**別パッケージ**です。これを忘れると実行時に `Template action with ID 'publish:gitlab' is not registered` エラーになります。

```bash
yarn --cwd packages/backend add @backstage/plugin-scaffolder-backend-module-gitlab
```

   `packages/backend/src/index.ts` の scaffolder 関連の並びに 1 行追加します。

```typescript
backend.add(import('@backstage/plugin-scaffolder-backend-module-gitlab'));
```

2. `my-idp` 直下にテンプレート用ディレクトリを作成します。

```bash
mkdir -p examples/template-gitlab/skeleton
```

3. `examples/template-gitlab/template.yaml` を作成します。

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: nodejs-service-gitlab
  title: Node.js サービス(GitLab)
  description: Node.js サービスの雛形リポジトリを GitLab に作成し、カタログに登録します
  tags:
    - handson
    - nodejs
spec:
  owner: guests
  type: service

  parameters:
    - title: サービスの基本情報
      required:
        - name
      properties:
        name:
          title: サービス名
          type: string
          description: リポジトリ名にもなります(例 my-new-service)
          ui:autofocus: true
        description:
          title: 説明
          type: string
          description: このサービスは何をするものか
    - title: 作成先リポジトリ
      required:
        - repoUrl
      properties:
        repoUrl:
          title: リポジトリの場所
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - gitlab.com

  steps:
    - id: fetch
      name: スケルトンを生成
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          destination: ${{ parameters.repoUrl | parseRepoUrl }}

    - id: publish
      name: GitLab にリポジトリを作成
      action: publish:gitlab
      input:
        repoUrl: ${{ parameters.repoUrl }}
        defaultBranch: main
        settings:
          description: ${{ parameters.description }}
          visibility: private

    - id: register
      name: カタログに登録
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

  output:
    links:
      - title: 作成されたリポジトリ
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: カタログで開く
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}
```

4. スケルトンを作成します。`fetch:template` はスケルトン内の `${{ values.xxx }}` を入力値で置換します。

`examples/template-gitlab/skeleton/catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name }}
  description: ${{ values.description }}
  annotations:
    gitlab.com/project-slug: ${{ values.destination.owner }}/${{ values.destination.repo }}
spec:
  type: service
  lifecycle: experimental
  owner: guests
```

`examples/template-gitlab/skeleton/README.md`:

```markdown
# ${{ values.name }}

${{ values.description }}

このリポジトリは Backstage の Software Template から自動生成されました。
```

`examples/template-gitlab/skeleton/index.js`:

```javascript
console.log('Hello from ${{ values.name }}');
```

5. テンプレートをカタログに登録します。`app-config.yaml` の `catalog.locations` に追記します(既存の `examples/template/template.yaml` のエントリの下に並べる)。

```yaml
    - type: file
      target: ../../examples/template-gitlab/template.yaml
      rules:
        - allow: [Template]
```

6. 再起動して実行します。

```bash
export GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
yarn start
```

7. ブラウザで **Create...** を開くと「Node.js サービス(GitLab)」が増えているので選択し、フォームに入力します。

![Create画面に追加されたGitLabテンプレート](images/template-gitlab-card.png)


| 入力欄 | 値の例 |
|---|---|
| サービス名 | `order-service` |
| 説明 | `注文を管理するサービス` |
| Owner(リポジトリの場所) | `handson-idp-catalog`(グループパス) |
| Repository | `order-service` |

8. **Review → Create** で実行し、各ステップが緑色で完了するのを見届けます。

✅ **確認ポイント**
- GitLab に `handson-idp-catalog/order-service` リポジトリが新規作成され、README / index.js / catalog-info.yaml が入っている(下図。コミッターが Scaffolder になっている点にも注目)

![テンプレートから自動作成されたGitLabリポジトリ](images/gitlab-order-service-repo.png)

- 実行結果画面のリンクからリポジトリとカタログエンティティに飛べる
- Catalog に `order-service` が登録されている

> 💡 **ここで学んだこと**: Scaffolder は `steps` に並べたアクション(`fetch:template` → `publish:gitlab` → `catalog:register`)を順に実行するワークフローエンジン。テンプレートに CI 設定(`.gitlab-ci.yml`)や lint 設定を仕込んでおけば、組織標準の「ゴールデンパス」を全開発者に配布できる。

---

### 演習 6(任意): エンティティ間の関係を定義してグラフで見る

**目的**: Component 単体ではなく、System やサービス間依存を表現してカタログを「地図」にする。

#### 手順

1. GitLab の `sample-service` リポジトリの `catalog-info.yaml` を編集し、System 定義と依存関係を追記します(`---` 区切りで 1 ファイルに複数エンティティを書けます)。

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: handson-system
  description: ハンズオンで作ったシステム一式
spec:
  owner: guests
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: sample-service
  description: ハンズオン用のサンプルサービス
  annotations:
    gitlab.com/project-slug: handson-idp-catalog/sample-service
  tags:
    - handson
spec:
  type: service
  lifecycle: experimental
  owner: guests
  system: handson-system
  dependsOn:
    - component:sample-service-2
```

2. コミット後、自動ディスカバリーの次回スキャンを待つ(またはバックエンド再起動)。

✅ **確認ポイント**: `sample-service` のエンティティページ下部の **Relations** に依存関係が表示され、左メニューの **Catalog Graph** でシステム構成がグラフとして可視化される(下図: 所有 ownerOf・所属 hasPart・依存 dependsOn の3種類の関係が1つの地図になっている)。

![Catalog Graphで可視化されたエンティティ間の関係](images/catalog-graph-relations.png)


> 💡 **ここで学んだこと**: カタログは単なる一覧ではなく、`system` / `dependsOn` / `providesApis` などでエンティティ同士を関係付けた「組織のソフトウェア地図」になる。

---

## 4. 習得事項のまとめ

### 4.1 このハンズオンで触れた要素

| 要素 | 演習 | ポイント |
|---|---|---|
| `@backstage/create-app` | 演習 1 | 雛形生成。app / backend のモノレポ構成 |
| `app-config.local.yaml` | 演習 2〜 | ローカル専用の設定上書き。秘密情報は `${環境変数}` 参照 |
| `integrations.gitlab` | 演習 2 | PAT(api / read_repository / write_repository)で GitLab API 連携 |
| `catalog-info.yaml` | 演習 3 | エンティティ定義。リポジトリ側に置くのが Backstage 流 |
| Register Existing Component | 演習 3 | URL 指定の手動登録(スモールスタート向け) |
| `plugin-catalog-backend-module-gitlab` | 演習 4 | グループ配下を定期スキャンする Entity Provider |
| `catalog.providers.gitlab` | 演習 4 | host / group / branch / schedule を宣言 |
| Software Template | 演習 5 | parameters(フォーム定義)+ steps(アクション実行) |
| `publish:gitlab` アクション | 演習 5 | GitLab へのリポジトリ作成と初回 push |
| System / dependsOn | 演習 6 | エンティティ間のリレーション表現 |

### 4.2 トラブルシューティング

| 症状 | 原因と対処 |
|---|---|
| `create-app` や `yarn install` が失敗する | Node.js のバージョンを確認(`node -v` が 22 系か)。ネイティブモジュールのビルドエラーの場合は Node を LTS に揃えると解消することが多い |
| `yarn: command not found` | `corepack enable` を実行(Git Bash を管理者権限で開き直す) |
| 画面は出るが一部機能がおかしい(カタログが空、Create... が出ない等) | 一部プラグインだけ初期化に失敗した部分故障。起動ログに `Plugin initialization complete` が出ているか確認する。代わりに `Plugin 'xxx' threw an error during startup` が出ていたら直後のエラーが原因(まずは再起動、直らなければエラー内容を読む) |
| 画面は出るがカタログが空(0件)/ ログに `NODE_MODULE_VERSION` エラー | Node バージョンの混在。ネイティブモジュール(better-sqlite3)がインストール時と別バージョンの Node で実行されている。**新しいターミナルを開いて `node -v` が想定バージョンか確認**してから `yarn start` する(nvm 等で切り替えた場合、切り替え前から開いていたターミナルは古い Node を使い続ける)。それでも直らない場合は `node_modules` を削除して `yarn install` |
| 起動時に `EADDRINUSE` | ポート 3000 / 7007 が使用中。既存プロセスを終了するか PC を再起動 |
| カタログ登録時に 401 / 403 | PAT のスコープ不足(`api` が必須)か有効期限切れ。`GITLAB_TOKEN` を export したシェルと `yarn start` したシェルが同じかも確認 |
| カタログ登録時に 404 | URL の誤り(`/-/blob/main/catalog-info.yaml` の形式か)、または Private プロジェクトにトークン未設定でアクセスしている |
| 自動ディスカバリーで何も出てこない | ① `group` がフルパスか(サブグループは `parent/child` 形式) ② `branch: main` とリポジトリの実ブランチが一致するか ③ `schedule` の周期を待ったか ④ バックエンドログにエラーがないか |
| `publish:gitlab` で `not allowed to have the additional property "xxx"` | アクションの入力スキーマとの不一致(バージョンにより仕様が変わる)。`description` や `visibility` は `settings:` 配下に書く。使えるプロパティは画面上部 **Create... → Actions** タブで実機のスキーマを確認できる |
| `Template action with ID 'publish:gitlab' is not registered` | Scaffolder 用 GitLab モジュール(`@backstage/plugin-scaffolder-backend-module-gitlab`)の追加漏れ。演習 5 の手順 1 を実施する |
| `publish:gitlab` が 403 で失敗 | PAT に `write_repository` / `api` があるか、対象グループに対して自分が Owner / Maintainer か |
| 環境変数が読まれず `${GITLAB_TOKEN}` のまま扱われる | `export` と `yarn start` を同一ターミナルで実行する |

### 4.3 実務への応用

- **テンプレートの拡充**: `.gitlab-ci.yml`、Dockerfile、lint / test 設定をスケルトンに含め、組織標準を「デフォルトで正しい」状態で配る
- **カタログの運用ルール化**: 「全リポジトリに catalog-info.yaml 必須」を CI でチェックし、ディスカバリーと組み合わせて台帳を自動維持する
- **GitLab プラグイン導入**: コミュニティ製 [GitLab プラグイン](https://github.com/immobiliare/backstage-plugin-gitlab)で、エンティティページに MR・パイプライン・リリース情報を表示できる

---

## 5. 今後の学習ロードマップ

| 優先度 | トピック | 内容 |
|---|---|---|
| ★★★ | **TechDocs** | MkDocs ベースのドキュメントをエンティティページに統合(docs-like-code)。テンプレートに docs/ を含めると効果大 |
| ★★★ | **認証と組織情報の取り込み** | ゲストログインをやめ、GitLab OAuth でサインイン。GitLab の User / Group をカタログの User / Group エンティティとして取り込み、`owner` を実組織と一致させる |
| ★★☆ | **本番環境へのデプロイ** | Docker イメージ化 + PostgreSQL + Kubernetes(または各種 PaaS)での常設運用。app-config の本番レイヤ設計 |
| ★☆☆ | **カスタムプラグイン開発** | `yarn new` でプラグインの雛形を作り、社内システム(監視・コスト・チケット等)をポータルに統合 |

### 参考リンク

- [Backstage 公式ドキュメント(Getting Started)](https://backstage.io/docs/getting-started/)
- [GitLab Locations(インテグレーション設定)](https://backstage.io/docs/integrations/gitlab/locations/)
- [GitLab Discovery(自動ディスカバリー)](https://backstage.io/docs/integrations/gitlab/discovery/)
- [Writing Templates(テンプレートの書き方)](https://backstage.io/docs/features/software-templates/writing-templates)
- [Builtin actions(publish:gitlab などの一覧)](https://backstage.io/docs/features/software-templates/builtin-actions)
- [System Model(エンティティモデルの解説)](https://backstage.io/docs/features/software-catalog/system-model)
- [TechDocs](https://backstage.io/docs/features/techdocs/)
- [コミュニティ GitLab プラグイン(immobiliare)](https://github.com/immobiliare/backstage-plugin-gitlab)
- [Backstage デモサイト(完成形のイメージ確認に)](https://demo.backstage.io/)

---

> 📁 本教材の図は `drawio/` フォルダに DrawIO ソース(.drawio)があります。draw.io で開いて編集し、PNG を再エクスポートできます。
