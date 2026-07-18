# Agent Skills 標準 ハンズオン — スキルを自作して「作成者側」に回る

---

## 1. Agent Skills 標準の概要

**Agent Skills** は、AIエージェントに「手順的な知識」を教えるためのオープン標準（[agentskills.io](https://agentskills.io)）です。実体は `SKILL.md` という1つのMarkdownファイル（＋任意の補助ファイル）で、Claude Code / GitHub Copilot / Cursor など複数プラットフォームで動きます。Superpowers も grill-me も、中身はこの標準に沿ったスキルの集合です。

### 押さえるべき5つの概念

1. **SKILL.md**: YAMLフロントマター（いつ使うかをClaudeに伝える）＋ Markdown本文（実行時の指示）
2. **2つの起動方法**: ユーザーが `/スキル名` で明示起動、または description にマッチした時にClaudeが自動起動
3. **遅延読み込み（Progressive Disclosure）**: 通常は description だけがコンテキストに載り、本文は起動時に初めて読み込まれる。CLAUDE.md と違い、使わない限りトークンをほぼ消費しない
4. **格納場所がスコープを決める**:

   | 場所 | パス | 適用範囲 |
   |---|---|---|
   | 個人 | `~/.claude/skills/<スキル名>/SKILL.md` | 自分の全プロジェクト |
   | プロジェクト | `.claude/skills/<スキル名>/SKILL.md` | そのプロジェクトのみ（Git共有可） |
   | プラグイン | `<plugin>/skills/<スキル名>/SKILL.md` | プラグイン導入先 |

5. **カスタムコマンドとの統合**: 旧 `.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` はどちらも `/deploy` になる（現在はスキルが推奨。補助ファイル等の追加機能があるため）

ディレクトリ名がそのままコマンド名になります（`.claude/skills/deploy-staging/` → `/deploy-staging`）。

### SKILL.md フロントマター項目一覧

このハンズオン（演習1〜6）で登場するフロントマター項目です。`description` 以外はすべて任意項目で、必要なものだけ組み合わせます。

| 項目 | 必須/任意 | 説明 | 登場演習 |
|---|---|---|---|
| `description` | 必須 | 「何をするか＋いつ使うか（トリガーになる言い回し）」を書く。Claudeはこれだけを見て自動起動を判断する | 演習1 |
| `argument-hint` | 任意 | `/`メニューに表示される引数のヒント文字列（例: `"[type: feat/fix/docs など（省略可）]"`） | 演習3 |
| `disable-model-invocation` | 任意 | `true`にするとClaudeによる自動起動を無効化し、`/スキル名`での明示起動のみに限定する。commit・deploy・通知送信など副作用のある操作向け | 演習3 |
| `user-invocable` | 任意 | `false`にするとユーザーの`/`起動を無効化し、Claudeの自動起動のみに限定する。Claudeにだけ使わせたい背景知識向け | 演習3（比較表として紹介） |
| `allowed-tools` | 任意 | 許可確認なしで実行できるツール・コマンドパターンを指定（例: `Bash(git diff *) Bash(git log *)`） | 演習4 |
| `context` | 任意 | `fork`を指定すると、会話履歴から隔離されたサブエージェントでスキル本文が実行される。調査系スキル向け | 演習5 |
| `agent` | 任意 | `context: fork`時に使うサブエージェントを指定（`Explore` / `Plan` / `general-purpose` / 自作サブエージェント等） | 演習5 |

起動制御の組み合わせをまとめると次の通りです（演習3で扱う表の再掲）。

| フロントマター | 自分が起動 | Claudeが起動 | 用途 |
|---|---|---|---|
| （デフォルト） | ○ | ○ | 一般的なスキル |
| `disable-model-invocation: true` | ○ | × | 副作用のあるワークフロー |
| `user-invocable: false` | × | ○ | 背景知識 |

### SKILL.md 本文で使える構文

フロントマター以外に、Markdown本文中で使える特殊構文です。

| 構文 | 説明 | 登場演習 |
|---|---|---|
| `` !`cmd` `` | スキル本文がClaudeに渡る**前**にシェルコマンドを実行し、結果を埋め込む（前処理）。行頭または空白の直後の`!`のみ認識される | 演習2 |
| `$ARGUMENTS` | ユーザーが `/スキル名 ...` に続けて渡した引数全体 | 演習5 |
| `$0`, `$1`... | 引数を空白区切りで分解した位置引数（`$0`が最初の1語） | 演習3 |
| `[file.md](file.md)` | 補助ファイルへのMarkdownリンク。Claudeが必要と判断したタイミングで読み込む（Progressive Disclosure） | 演習4 |

---

## 2. ハンズオンの概要

- 対象環境: Windows + VS Code + Claude Code（ターミナルは Git Bash 想定）
- プロジェクト作成先: `/c/dev/handson-agent-skills`
- 所要時間目安: 2〜2.5時間（演習1〜6）
- 参照: [Claude Code Skills 公式ドキュメント](https://code.claude.com/docs/en/skills)（2026-07 時点の仕様に準拠）

### ゴールイメージ

このハンズオンを終えると、以下6つのスキルが手元にできあがり、Agent Skills標準の主要機能をひと通り自作・動作確認済みの状態になります。

| # | スキル名 | 学ぶフロントマター/構文 | できるようになること |
|---|---|---|---|
| 1 | `summarize-changes` | `description` | 最小構成のスキルを作り、明示起動・自動起動を切り分けられる |
| 2 | `summarize-changes`（改） | `` !`cmd` `` | シェルコマンドの実行結果を前処理で確実にプロンプトへ埋め込める |
| 3 | `commit-msg` | `argument-hint`, `$0`, `disable-model-invocation` | 引数を受け取り、副作用のある操作の起動タイミングを制御できる |
| 4 | `ai-code-review` | `allowed-tools`, 補助ファイル | SKILL.mdを概要に絞り、詳細を別ファイルに逃がす設計ができる |
| 5 | `deep-research` | `context: fork`, `agent` | 調査タスクをサブエージェントに隔離実行させられる |
| 6 | （`ai-code-review`の評価） | skill-creator | with/withoutスキル比較でスキルの効果を定量評価できる |

### 学べること（全体像）

- SKILL.md の構造（フロントマター＋本文）と、格納場所によるスコープの違い
- 明示起動（`/スキル名`）と自動起動（description マッチ）の使い分け
- Progressive Disclosure（遅延読み込み）によるトークン消費の抑制
- 副作用のある操作の起動制御（`disable-model-invocation` / `user-invocable`）
- スキルの評価（skill-creator）と、チーム・組織への配布方法

---

## 3. ハンズオンの手順

### 3.1 事前準備

```bash
# 確認
claude --version    # Claude Code
git --version

# プロジェクト作成
mkdir -p /c/dev/handson-agent-skills
cd /c/dev/handson-agent-skills
git init
code .
```

VS Code の統合ターミナルで `claude` を起動しておきます。スキルファイルの追加・編集はセッション中に自動検知されるため、**Claude Code を再起動せずに試せます**（新しい skills ディレクトリ自体を初めて作った場合のみ再起動が必要）。

---

### 3.2 演習1: 最小のスキルを作る（15分）

「未コミットの変更を要約する」スキルを作ります。まず骨格だけ。

```bash
mkdir -p .claude/skills/summarize-changes
```

`.claude/skills/summarize-changes/SKILL.md` を VS Code で作成:

```yaml
---
description: 未コミットの変更を要約し、リスクを指摘する。ユーザーが「何を変更した？」「コミットメッセージを作って」「差分をレビューして」と言ったときに使用する。
---

git diff HEAD を実行して変更内容を取得し、以下を出力してください。

1. 変更の要約（2〜3行の箇条書き）
2. 気づいたリスク（エラー処理の欠落、ハードコード、テスト未更新など）

差分が空なら「未コミットの変更はありません」と答えてください。
```

**動作確認**（適当なファイルを1つ編集してから）:

```text
/summarize-changes        ← 明示起動
```

```text
何を変更したか教えて      ← 自動起動（descriptionにマッチ）
```

✅ 確認ポイント:

- `/` を入力すると `summarize-changes` が候補に出る
- 自動起動の場合、応答の中でスキルが読み込まれたことが表示される
- `What skills are available?` と聞くと一覧に出る

**ここで学んだこと**: description は「何をするか＋いつ使うか（トリガーになる言い回し）」を書く。Claudeはこれだけを見て起動判断をする。

---

### 3.3 演習2: 動的コンテキスト注入（15分）

演習1では「git diff を実行して」とClaudeに指示しましたが、`` !`コマンド` `` 構文を使うと、**スキル本文がClaudeに渡る前に**シェルコマンドが実行され、出力が埋め込まれます。

SKILL.md を書き換え:

```yaml
---
description: 未コミットの変更を要約し、リスクを指摘する。ユーザーが「何を変更した？」「コミットメッセージを作って」「差分をレビューして」と言ったときに使用する。
---

## 現在の変更

!`git diff HEAD`

## 指示

上記の変更を2〜3行で要約し、リスク（エラー処理の欠落、ハードコード、
テスト未更新など）を指摘してください。差分が空なら「未コミットの変更は
ありません」と答えてください。
```

再度 `/summarize-changes` を実行。

✅ 確認ポイント: Claudeがツール（Bash）を呼ばずに即座に要約を返す。差分は事前処理で注入済みのため、「Claudeが実行する」のではなく「実行結果を読む」動きになる。

**ここで学んだこと**: `` !`cmd` `` は前処理。Claudeの解釈を挟まずに事実データを渡せるため、確実性が上がる。行頭または空白の直後の `!` のみ認識される点に注意。

---

### 3.4 演習3: 引数と起動制御（20分）

コミットメッセージを生成するスキルで、引数（`$ARGUMENTS`, `$0`）と起動制御（`disable-model-invocation`）を学びます。

```bash
mkdir -p .claude/skills/commit-msg
```

`.claude/skills/commit-msg/SKILL.md`:

```yaml
---
description: ステージ済みの変更からConventional Commits形式の日本語コミットメッセージを生成する
argument-hint: "[type: feat/fix/docs など（省略可）]"
disable-model-invocation: true
---

## ステージ済みの変更

!`git diff --cached`

## 指示

上記の変更に対する Conventional Commits 形式のコミットメッセージを
日本語で1つ提案してください。type の指定があれば従うこと: $0

- 1行目: type(scope): 要約（50文字以内）
- 空行を挟んで本文（変更理由を2〜3行）
- git commit は実行しない。メッセージの提案のみ行う
```

**動作確認**:

```bash
# Git Bash側で変更をステージしてから
git add -A
```

```text
/commit-msg          ← 引数なし
/commit-msg feat     ← $0 に "feat" が入る
```

✅ 確認ポイント:

- `/` メニューで `[type: feat/fix/docs など（省略可）]` のヒントが表示される
- 「コミットメッセージ作って」と話しかけても**このスキルは自動起動しない**（`disable-model-invocation: true` のため。代わりに演習1のスキルが反応するはず）

**ここで学んだこと**: 副作用のある操作や、実行タイミングを自分で決めたい操作は `disable-model-invocation: true` にする（deploy、commit、通知送信など）。逆に、Claudeにだけ使わせたい背景知識は `user-invocable: false` にする。

---

### 3.5 演習4: 補助ファイルで本格スキルを作る（30分）

Agent Skills の本領は **Progressive Disclosure**（必要なときだけ詳細を読み込む）です。AI生成コードのレビュースキルを、チェックリストを別ファイルに分離して作ります。

```bash
mkdir -p .claude/skills/ai-code-review
```

`.claude/skills/ai-code-review/SKILL.md`:

```yaml
---
description: AI駆動開発の観点でコード変更をレビューする。AI生成コードのレビュー、変更の影響範囲確認、リリース前チェックを求められたときに使用する。
allowed-tools: Bash(git diff *) Bash(git log *)
---

## 現在の変更

!`git diff HEAD`

## 指示

上記の変更を [checklist.md](checklist.md) の観点でレビューし、
観点ごとに OK / NG / 対象外 を判定して根拠を1行で示してください。
NG があればブロッカーとして冒頭にまとめてください。
```

`.claude/skills/ai-code-review/checklist.md`（AI駆動開発6つのポイントを観点化）:

```markdown
# AIレビューチェックリスト

1. 影響範囲: この変更は上流成果物（仕様・設計文書）と矛盾しないか
2. 仕様との対応: 変更内容は仕様・タスクのどれに対応するか特定できるか
3. テスト: 変更されたロジックに対応するテストがあるか
4. エラー処理: 異常系（空入力・失敗パス）が考慮されているか
5. ハードコード: 環境依存値・マジックナンバーが埋め込まれていないか
6. 最小性: 要求されていない機能・過剰な抽象化が混入していないか（YAGNI）
```

**動作確認**:

```text
この変更をAIレビューして
```

✅ 確認ポイント:

- Claudeが checklist.md を必要に応じて読み込み、6観点で判定する
- `allowed-tools` により `git diff` / `git log` は許可確認なしで実行される

**ここで学んだこと**: SKILL.md は「概要とナビゲーション」に徹し（500行以内推奨）、詳細は補助ファイルへ。SKILL.md から必ずリンクして「何が書いてあるか・いつ読むか」を伝える。スクリプトを同梱する場合は `${CLAUDE_SKILL_DIR}/scripts/xxx.py` で参照する。

```text
ai-code-review/
├── SKILL.md        # 必須。概要とナビゲーション
├── checklist.md    # 必要時に読み込まれる参照資料
└── scripts/        # （任意）実行スクリプト
```

---

### 3.6 演習5: サブエージェント実行（15分）

`context: fork` を付けると、スキルは会話履歴から隔離されたサブエージェントで実行されます。調査系スキルに向いています。

```bash
mkdir -p .claude/skills/deep-research
```

`.claude/skills/deep-research/SKILL.md`:

```yaml
---
description: コードベース内のトピックを徹底調査する
context: fork
agent: Explore
---

$ARGUMENTS について徹底的に調査してください。

1. Glob / Grep で関連ファイルを特定する
2. コードを読んで分析する
3. ファイル参照付きで発見事項を要約する
```

**動作確認**（少しコードのあるリポジトリで）:

```text
/deep-research エラーハンドリングの実装方法
```

✅ 確認ポイント: サブエージェントが起動し、要約だけがメイン会話に返る（調査の途中経過でメインのコンテキストが消費されない）。

**ここで学んだこと**: `context: fork` はSKILL.md本文がそのままサブエージェントへのプロンプトになる。「ガイドラインだけ」のスキルに付けると、タスクがなく空振りするので注意。`agent` には `Explore` / `Plan` / `general-purpose` / 自作サブエージェントを指定できる。

---

### 3.7 演習6: スキルの評価と改善 — skill-creator（20分）

スキルは「起動したか」と「意図どおりの出力か」を分けて評価します。公式の skill-creator プラグインが評価ループを自動化してくれます。

```text
/plugin install skill-creator@claude-plugins-official
/reload-plugins
```

（marketplace が見つからない場合: `/plugin marketplace add anthropics/claude-plugins-official`）

**評価の実行**:

```text
skill-creatorを使って、ai-code-review スキルを評価してください
```

skill-creator は次を行います: テストケースの作成（`evals/evals.json`）、サブエージェントでの隔離実行、合否判定（`grading.json`）、スキルあり/なしのベンチマーク比較（`benchmark.json`）、description のチューニング（起動すべき/すべきでないプロンプトでのヒット率測定）。

✅ 確認ポイント: 「スキルなし」との比較で、パス率の改善がトークン・時間のオーバーヘッドに見合うかを確認する。

**ここで学んだこと**: スキル作成は書いて終わりではなく、Spec Kit の analyze と同様に「評価ゲート」を挟んで改善する。

---

## 4. 習得事項のまとめ

### 4.1 配布 — チーム・組織への展開

作ったスキルの共有方法は3段階あります。

1. **プロジェクト共有**: `.claude/skills/` を Git にコミットするだけ。リポジトリを clone した全員が使える（今回の演習スキルはこの状態）

   ```bash
   git add .claude && git commit -m "feat: AIレビュー等のスキルを追加"
   ```

2. **プラグイン化**: `skills/` ディレクトリを持つプラグインとしてパッケージし、`plugin-name:skill-name` の名前空間で配布
3. **マーケットプレイス**: プラグインをカタログ化したGitリポジトリを作り、チームは `/plugin marketplace add <repo>` で一括導入

社内標準（レビュー観点・輸出管理チェック・ASPICE成果物テンプレート等）をスキル化してマーケットプレイスで配る、というのが組織展開の定石です。

### 4.2 トラブルシューティング

| 症状 | 対処 |
|---|---|
| スキルが自動起動しない | description にユーザーが実際に言いそうなキーワードを入れる。`What skills are available?` で認識確認。`/スキル名` で直接起動できるかも確認 |
| 起動しすぎる | description を具体的に絞るか `disable-model-invocation: true` |
| `/スキル名` は動くが自動起動しない | フロントマターYAMLの構文エラーの可能性（壊れていると description なしで読み込まれる）。`claude --debug` でパースエラー確認 |
| 新しいスキルが認識されない | 既存の skills ディレクトリ内なら即反映。ディレクトリ自体を新規作成した場合は Claude Code を再起動 |
| description が途切れる | スキル一覧のコンテキスト予算（モデルの1%）を超過。`/doctor` でコスト確認。description は先頭に主要ユースケースを書く（1,536文字で切られる） |
| `!` コマンドが実行されない | `!` が行頭か空白直後にあるか確認。`KEY=!`cmd`` のような位置では動かない |

### 4.3 振り返り

このハンズオンで、Agent Skills 標準の主要素をすべて触りました: SKILL.md の構造、明示/自動起動、動的コンテキスト注入、引数、起動制御、補助ファイル（Progressive Disclosure）、`allowed-tools`、`context: fork`、評価（skill-creator）、配布。

---

## 5. 今後の学習ロードマップ

### 次のステップ

1. Superpowers・grill-me の SKILL.md を読む（教材として最良。スキル設計の実例が数十個ある）
2. 業務の定型手順（毎回チャットに貼っているチェックリストや手順書）を1つスキル化する
3. `hooks` フロントマターや自作サブエージェントとの組み合わせに進む

### 参考リンク

- [Claude Code Skills 公式ドキュメント](https://code.claude.com/docs/en/skills)
- [Agent Skills オープン標準](https://agentskills.io)
- [スキル作成のベストプラクティス](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [skill-creator プラグイン](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/skill-creator)
- [obra/superpowers（実例集として）](https://github.com/obra/superpowers)
