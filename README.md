# graph-engineering-design

リポジトリの実運用ワークフローを**明示的な実行グラフ**として設計するための Claude Code プラグイン（Skill）。

ノード・エッジ・共有状態・ルーティング・checkpoint を列挙し、暗黙エッジを狩り、人間とのレビューループで
`graph-design.md`（設計根拠と決定台帳）と `graph.md`（人間向け投影）を作成します。
設計のみを行い、skill / rule / script などのハーネス実体には人間の GO まで触れません。

## インストール

```
/plugin marketplace add jpwstu/graph-engineering-design
/plugin install graph-engineering-design@graph-engineering-design
```

## 使い方

「このリポジトリで○○をするための Graph Engineering を設計したい」と依頼すると起動します。

## 構成

```
.claude-plugin/
  marketplace.json   # マーケットプレイス定義
  plugin.json        # プラグイン定義
skills/
  graph-engineering/
    SKILL.md
    references/
```

## ライセンス

MIT
