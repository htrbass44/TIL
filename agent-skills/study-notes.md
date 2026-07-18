# Agent Skills ハンズオン 学習記録

- 実施日: 2026-07-17 〜 2026-07-18
- 環境: Windows + VS Code + Claude Code（Git Bash）
- 参照手順書: [agent-skills-handson.md](agent-skills-handson.md)

---

## 事前準備

- `git init` でリポジトリ作成
- `git --version` / `claude --version` 確認済み

---

## 演習1: 最小のスキルを作る（summarize-changes）

`.claude/skills/summarize-changes/SKILL.md` を作成。description のみを指定した最小構成のスキル。

```yaml
---
description: 未コミットの変更を要約し、リスクを指摘する。ユーザーが「何を変更した？」「コミットメッセージを作って」「差分をレビューして」と言ったときに使用する。
---

git diff HEAD を実行して変更内容を取得し、以下を出力してください。
...
```

### 動作確認結果

- `/summarize-changes` で明示起動 → ✅ 成功
- `What skills are available?` で一覧表示 → ✅ 成功

### トラブルと対処

1. **ネストしたGitリポジトリ**: `.claude/skills/summarize-changes/` ディレクトリ内で誤って `git init` を実行してしまい、外側の `git add` が `does not have a commit checked out` エラーで失敗。
   - 原因: サブディレクトリ内に `.git` ができ、外側のGitがそこを「コミットのないsubmodule」として扱おうとした
   - 対処: 中身が空リポジトリ（コミットなし）であることを確認後、`.claude/skills/summarize-changes/.git` を削除して解消
2. **`git diff HEAD` が初回コミット前に失敗**: リポジトリにコミットが1つもない状態では `HEAD` が存在せず `fatal: ambiguous argument 'HEAD'` になる。
   - 対処: 演習1の成果物一式（`.claude/`, `README.md`, `agent-skills-handson.md`）を初回コミットとして commit。以降 `git diff HEAD` が正常動作するようになった。
   - **学び**: 手順書の SKILL.md は「初回コミット前」を想定していない。実運用では `git diff HEAD 2>/dev/null || git diff --stat` のような分岐や、先に初回コミットを済ませる運用が現実的。

---

## 演習2: 動的コンテキスト注入（`` !`cmd` `` 構文）

SKILL.md を書き換え、`` !`git diff HEAD` `` によりスキル本文がClaudeに渡る前にコマンドを実行・結果を埋め込む形に変更。

### 動作確認結果

- ✅ Claudeが Bash ツールを呼ばずに、事前注入済みの diff 結果を読むだけで要約を返した
- 演習1（Claudeが自分で `git diff HEAD` を実行）との違いを体感できた。前処理により確実に同じデータが渡る

---

## 演習3: 引数と起動制御（commit-msg）

`.claude/skills/commit-msg/SKILL.md` を作成。`argument-hint`、`$0`（第1引数）、`disable-model-invocation: true` を使用。

```yaml
---
description: ステージ済みの変更からConventional Commits形式の日本語コミットメッセージを生成する
argument-hint: "[type: feat/fix/docs など（省略可）]"
disable-model-invocation: true
---
```

### 学んだこと

- `/commit-msg feat` の `feat` は `$0` に代入され、本文中の `$0` の位置に埋め込まれる
- `feat` は Conventional Commits 規約の変更種別プレフィックス（新機能追加の意味）。他に `fix`（バグ修正）、`docs`（ドキュメントのみ）、`refactor`、`test`、`chore` などがある
- `disable-model-invocation: true` により、「コミットメッセージ作って」等の自然言語では自動起動しない（副作用のある操作は明示起動のみに絞る設計）

| フロントマター | 自分が起動 | Claudeが起動 | 用途 |
|---|---|---|---|
| （デフォルト） | ○ | ○ | 一般的なスキル |
| `disable-model-invocation: true` | ○ | × | 副作用のあるワークフロー |
| `user-invocable: false` | × | ○ | 背景知識 |

---

## 演習4: 補助ファイルで本格スキル（ai-code-review）

`.claude/skills/ai-code-review/SKILL.md` ＋ `checklist.md`（AIレビュー観点6つ）を作成。`allowed-tools` で `git diff` / `git log` を許可確認なしで実行できるように設定。

### 動作確認結果

「この変更をAIレビューして」で自動起動 → ✅ 成功。`checklist.md` を参照し、6観点（影響範囲・仕様との対応・テスト・エラー処理・ハードコード・最小性）でOK/NG/対象外を判定。

**実際に検出されたNG:**
1. README.mdの見出し「演習2の実行」と、同じコミットで追加された`commit-msg`スキル（演習3の内容）が対応していない → ドキュメントとコードの乖離を的確に検出
2. `commit-msg/SKILL.md` に「ステージ済み差分が空の場合」の分岐指示が欠落（`summarize-changes` には同等の記述があるのに不統一）

### 学んだこと

- SKILL.mdは「概要とナビゲーション」に徹し、詳細（チェックリスト等）は補助ファイルに分離するProgressive Disclosureの効果を体感
- `allowed-tools` により、許可プロンプトなしで特定コマンドだけ自動実行できる
- レビュー観点そのものが「AI駆動開発の品質チェック」を体現しており、演習中の教材自身の粗（ドキュメント不整合）を発見する実例になった

---

## 演習5: サブエージェント実行（deep-research）

`.claude/skills/deep-research/SKILL.md` を作成。`context: fork` と `agent: Explore` を指定。

```yaml
---
description: コードベース内のトピックを徹底調査する
context: fork
agent: Explore
---
```

### 動作確認結果

`/deep-research エラーハンドリングの実装方法` を実行 → ✅ サブエージェントが起動し、Glob/Grepでリポジトリ全体を調査、要約のみがメイン会話に返却された（途中経過でメインのコンテキストが消費されない動きを確認）。

このリポジトリには実装コードが存在しないため「該当なし」という結果になったが、サブエージェントは単に「なし」で終わらせず、なぜ該当しないか・関連語がどこに登場するか・このクエリ自体が手順書のサンプルクエリであることまで分析して報告した。

### 学んだこと

- `context: fork` はSKILL.md本文がそのままサブエージェントへのプロンプトになる
- 調査系タスクに向いており、メインの会話コンテキストを消費せずに深い調査ができる
- 実際にコードを含むリポジトリで試すとより効果が体感しやすい（手順書の注記通り）

---

## 演習6: skill-creatorによる評価

`/plugin install skill-creator@claude-plugins-official` を **user scope**（自分の全プロジェクトで使える個人ツールとして）でインストール。「skill-creatorを使って、ai-code-review スキルを評価してください」と依頼。

### 評価フロー（自動化された内容）

1. スクラッチパッドに3つのテストシナリオ用ミニGitリポジトリを作成
   - `docs-only-clean-change`: README軽微修正（誤検知しないか）
   - `missing-tests-and-error-handling`: テスト・異常系未対応の新規関数追加
   - `hardcoded-secret-contradicts-spec`: APIキーのハードコード＋仕様矛盾
2. `.claude/skills/ai-code-review/evals/evals.json` にテストケース定義を作成
3. 各シナリオを **with_skill（スキルあり）** / **without_skill（素のClaude Code）** で並列実行（6バックグラウンドエージェント）
4. 各結果を `grading.json` で自動採点、`benchmark.json` / `benchmark.md` に集計
5. ブラウザで結果ビューアを起動し、目視レビュー → `feedback.json` として保存

### 結果

| Eval | シナリオ | With Skill | Without Skill |
|---|---|---|---|
| 1 | READMEのみの軽微な変更 | 100% | 50% |
| 2 | テスト・エラー処理欠落のコード変更 | 100% | 66.7% |
| 3 | 仕様矛盾＋APIキーのハードコード | 100% | 100% |

**全体**: With Skill 100% vs Without Skill 72.2%（delta +0.28）、時間は+41.6秒、トークンは-321（baselineの方がやや冗長）。

フィードバックレビューでは6件中5件が「問題なし」、1件が「OK」のコメントのみで、修正要望はなし。**結論としてスキルは現状維持**（SKILL.md / checklist.mdへの変更なし）。

### 学んだこと

- **スキルの効果が出ている箇所を見極める**: 今回のケースでは、スキルの価値は「欠陥の発見力そのもの」ではなく「6観点の構造化判定＋NGを冒頭にブロッカーとしてまとめる、という出力フォーマットの一貫性」にあった。baseline（スキルなし）でも欠陥自体は発見できていた
- **non-discriminatingなテストケースへの気づき**: Eval3（APIキーのハードコード）はあまりに問題が深刻・明白なため、スキルなしでも強く警告される → with/withoutの差が出ず、評価セットとして「スキルの有無を判別できていない」ケースだと明確に指摘された。中程度の深刻さのケースを増やすべきという改善提案も得られた
- **n=1の限界**: 各シナリオ1回ずつの実行のため、stddevは参考程度という限界も自己申告されていた（本来は複数回実行してばらつきを見るべき）
- **評価は「起動したか」と「意図どおりの出力か」を分けて見る**ことの実践例になった。スキルを作って終わりではなく、with/withoutの比較で「そのスキルが本当に効果を出しているか」「効果が薄いテストケースはどれか」まで定量的に確認できた

---

## 全体を通しての気づき

1. **初回コミット問題**: 演習1のSKILL.mdは新規リポジトリ（コミット0件）を想定しておらず、`git diff HEAD` がエラーになった。ハンズオンを新規リポジトリで行う場合は、先に空コミットまたは初回コミットを作っておくとスムーズ。
2. **作業ディレクトリの取り違え注意**: スキルのサブディレクトリ内で誤って `git init` すると、ネストしたリポジトリ（gitlink問題）が発生し `git add` が失敗する。VS Code統合ターミナルでは現在のディレクトリを都度確認する習慣が有効。
3. **`ai-code-review` スキルの実効性**: チェックリストによる機械的なレビューが、ハンズオン中に発生した実際のドキュメント不整合（README見出しと実装内容のズレ）を検出できた。AI駆動開発における「レビュー観点のスキル化」の有用性を実感。
4. **評価（演習6）で得た定量的な視点**: with/withoutスキル比較を行うことで、「スキルが本当に効いている部分」と「効いていない部分（non-discriminating なテストケース）」を切り分けられた。スキル作成は「作って終わり」ではなく、評価ゲートを挟んで初めて効果が定量的に確認できることを体感した。
