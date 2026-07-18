# AI駆動開発ツール比較 ― AI-DLC / Spec Kit / Superpowers / cc-sdd / grill-me

作成日: 2026-07-16

対象: AWS AI-DLC、GitHub Spec Kit、Superpowers（obra）、cc-sdd（gotalab）、grill-me

※「GitHub SuperPower」は obra/superpowers、「glill-me」は grill-me スキルとして整理。Superpowers・cc-sdd・grill-me はGitHub上で公開されている個人/コミュニティ製OSSであり、GitHub社製は Spec Kit のみ。

## 比較表

| | 提供元 | 実体・レイヤー | 強み | 学習コスト |
|---|---|---|---|---|
| **AI-DLC** | AWS | 方法論（プロセス規範） | 組織導入の思想・人の承認ゲート設計。ツール非依存 | 中（概念中心） |
| **Spec Kit** | GitHub公式 | SDDツールキット | 成果物チェーン（spec→plan→tasks）とanalyzeによる整合性検査。エコシステム最大（extensions/presets） | 中 |
| **Superpowers** | obra（個人OSS、約9万Star） | 開発規律のスキルフレームワーク | TDD強制・2段階レビュー・サブエージェント・Git Worktreeで1〜2時間の自律開発を実現。SDDより広い「規律」全般をカバー | 高 |
| **cc-sdd** | gotalab（国産OSS） | SDDツール（Kiro互換） | 日本語完全対応、requirements→design→tasksの各段階で承認必須。Claude Code/Cursor/Copilot等マルチ対応 | 低〜中 |
| **grill-me** | Matt Pocock 他 | 単機能スキル | 計画・設計を1問ずつ徹底的に深掘り質問（`/speckit.clarify` の強化版に相当）。導入1分 | 低 |

## レイヤー整理

5つは競合ではなくレイヤーが異なる。

1. **方法論**: AI-DLC（プロセス規範。ツールを評価する際の判断軸として使う）
2. **SDDツール**: Spec Kit、cc-sdd（仕様→計画→タスク→実装の成果物チェーンを作る）
3. **規律・スキル**: Superpowers（実装プロセスの質を担保）、grill-me（要件深掘りの単機能）

## 習得の推奨順（AI-DLC・Spec Kit 実践済みの前提）

1. **Superpowers** ― 今の潮流（スキルベースのエージェント規律）を代表する存在で、SDDと相補的。Spec Kit が「成果物の質」を担保するのに対し、Superpowers は「実装プロセスの質」（TDD・レビュー・自律実行）を担保する。ASPICE対応の観点でも、検証戦略の弱さを補う考え方が学べる。一つ選ぶならこれ。
2. **grill-me** ― 学習というより即導入。要件深掘りの質が上がるため、Spec Kit の clarify 前に併用すると効果的。
3. **cc-sdd** ― Spec Kit と役割が重複するため優先度は低め。ただし日本語ネイティブ・軽量・Kiro互換のため、社内展開や日本語チームへの布教には Spec Kit より通しやすい場面がある。比較目的で半日触る価値はある。

AI-DLC は「ツール」ではなく思想のため、追加の習得というより、上記ツールを評価する際の判断軸として位置づける。

## 参考リンク

- [obra/superpowers](https://github.com/obra/superpowers)
- [Superpowers解説（Qiita）](https://qiita.com/nogataka/items/c2e73515e65533986421)
- [gotalab/cc-sdd（日本語README）](https://github.com/gotalab/cc-sdd/blob/main/docs/README/README_ja.md)
- [cc-sdd解説（Qiita）](https://qiita.com/tomada/items/6a04114fc41d0b86ffee)
- [grill-me（mattpocock/skills）](https://github.com/mattpocock/skills/tree/main/skills/productivity/grill-me)
- [grill-me試用記事（note）](https://note.com/masaru_furuya/n/nb13234e725a5)
