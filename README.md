# scala-skills

Scala 開発向けの Claude Code スキル集です。プラグインマーケットプレイスとして配布しています。

## インストール

Claude Code 上で以下を実行します。

```
/plugin marketplace add windymelt/scala-skills
/plugin install scala-skills@scala-skills
```

## スキル一覧

| スキル | 呼び出し | 説明 |
|--------|----------|------|
| [containerize](./skills/containerize/SKILL.md) | `/scala-skills:containerize` | Cloud Native Buildpacks で sbt プロジェクトを Dockerfile なしに OCI コンテナイメージ化する |

スキルはキーワード（「コンテナ化して」「Dockerイメージにして」など）でも自動発動します。

## 構成

```
.claude-plugin/
├── marketplace.json   # マーケットプレイス定義
└── plugin.json        # プラグイン定義
skills/
└── containerize/
    └── SKILL.md       # スキル本体
```

スキルを追加する場合は `skills/` 配下にディレクトリを作り、`SKILL.md` を置きます。
