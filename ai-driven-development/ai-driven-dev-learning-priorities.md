# 優先して学習すべき類似のSKILLS・ツール ― 追加推奨4選

作成日: 2026-07-16

前提: AI-DLC・Spec Kit 実践済み、Superpowers / grill-me / cc-sdd を学習候補として検討中。それらに追加する候補を優先度順に整理する。

## 1. Agent Skills 標準（SKILL.md）とプラグインの仕組み ― 最優先

個別ツールというより「基盤リテラシー」。Superpowers も grill-me も実体は SKILL.md 形式のスキルであり、この標準は Claude Code / Copilot / Cursor など14以上のプラットフォームに広がる。エコシステムは4,000超のスキル・2,500超のマーケットプレイスに成長（2026年5月時点）。

仕組み（スキルの書き方・プラグイン化・マーケットプレイスでの社内配布）を一度学べば、既存ツールの中身が読めるようになり、ASPICE観点のレビュースキルや輸出管理チェックスキルのような自作も可能。ツールの消費者から作成者側に回れる点で、投資対効果が最も高い。

**最初の一歩**: Agent Skills 標準を学んで簡単なスキルを1つ自作する。

## 2. OpenSpec ― ASPICE文脈なら Spec Kit の次に価値が高い

SDDフレームワークの主要3つ（Spec Kit / BMAD / OpenSpec）の一角。特徴は**変更管理**：変更提案を中心に、既存機能への変更を ADDED / MODIFIED / REMOVED のデルタで追跡し、実装前に監査可能な文書を残す。

「変更の影響範囲特定」「入力の記録」「合格基準」といったAI駆動開発のポイント、および ASPICE の SUP.10（変更依頼管理）と直結する思想。グリーンフィールド向けの Spec Kit をブラウンフィールド（既存システム改修）側から補完する。

## 3. BMAD-METHOD ― 概念として知っておく

Analyst / PM / Architect / UX / Scrum Master / Developer / QA など12以上の役割エージェントでアジャイルチームを模擬する、最も野心的なフレームワーク。各エージェントがスコープされたコンテキストを持ち、バージョン管理された成果物（PRD・アーキテクチャ文書・ストーリー）を次のエージェントに引き継ぐ。

AI-DLCの「人が承認する」構造を「役割エージェント間の引き継ぎ」に発展させた形として比較する価値があるが、重量級のため実務導入より概念学習向き。

## 4. Kiro（AWS） ― 軽く触る程度で十分

AWS純正のSDD統合IDE（Code OSS基盤）。cc-sdd が Kiro 互換のため、cc-sdd を触ればワークフロー概念はほぼカバーできる。AWS環境が主戦場になる場合は優先度を上げる。

## 学習マップ（3層×各1つ）

| 層 | 選択肢 | 深く押さえる推奨 |
|---|---|---|
| 方法論 | AI-DLC、BMAD-METHOD | AI-DLC（実践済み） |
| SDDツール | Spec Kit、OpenSpec、cc-sdd、Kiro | Spec Kit（実践済み）＋ OpenSpec |
| 規律・スキル基盤 | Superpowers、grill-me、Agent Skills標準 | **Agent Skills標準**（スキル自作まで） |

各層1つずつ深く押さえるのが効率的。次の一手として最も推奨するのは Agent Skills 標準の習得とスキルの自作。

## 参考リンク

- [Agent Skills公式ドキュメント](https://platform.claude.com/docs/ja/agents-and-tools/agent-skills/overview)
- [Claude Agent Skills完全ガイド（Qiita）](https://qiita.com/potofo/items/562610a6fb2f23f0e1ff)
- [Claude Code Marketplaceガイド](https://uravation.com/media/anthropic-marketplace-plugin-complete-guide-2026/)
- [BMAD vs Spec Kit vs OpenSpec（Reenbit）](https://reenbit.com/bmad-vs-spec-kit-vs-openspec-choosing-your-spec-driven-ai-framework/)
- [9 Best AI Tools for SDD 2026（MarkTechPost）](https://www.marktechpost.com/2026/05/08/9-best-ai-tools-for-spec-driven-development-in-2026-kiro-bmad-gsd-and-more-compare/)
- [OpenSpec比較ページ](https://openspec.pro/comparison/)
