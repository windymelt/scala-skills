---
name: containerize
description: >-
  ScalaプロジェクトをDockerfileを書かずにOCIコンテナイメージ化するスキル。
  Cloud Native Buildpacks（pack CLI + Paketo builder）と sbt-native-packager を使い、
  JVMメモリ設定の自動計算込みで実行可能なイメージを生成する。
  「コンテナ化して」「Dockerイメージにして」「OCIイメージを作って」「pack build」
  「Buildpackでビルド」などのキーワードで発動する。
---

# Scala プロジェクトの OCI コンテナ化（Cloud Native Buildpacks）

Dockerfile を書かずに、Cloud Native Buildpacks (CNB) で sbt プロジェクトを
OCI コンテナイメージに変換する。JVM のメモリパラメータ（`-Xmx` 等）は
起動時にコンテナのメモリ制限から自動計算されるため、手動チューニングが不要になる。

参考: https://blog.3qe.us/entry/2026/07/24/154004

## 前提条件

作業前に以下を確認する。欠けていればユーザーに導入を案内する。

1. Docker デーモンが動作していること（`docker info`）
2. `pack` CLI がインストールされていること（`pack version`）
   - 未導入なら https://buildpacks.io/docs/for-platform-operators/how-to/integrate-ci/pack/ を案内する
3. 対象が sbt プロジェクトであること（`build.sbt` の存在で Paketo sbt buildpack が発動する）
   - scala-cli / Mill プロジェクトはこのスキルの対象外。その旨を伝えること

## 手順

### 1. sbt バージョンの確認

`project/build.properties` を読んで sbt のメジャーバージョンを判定する。
sbt 1 系と 2 系で成果物のパスと必要な設定が異なる（手順 3 参照）。

### 2. sbt-native-packager の導入

`project/plugins.sbt` に追加する（既にあればスキップ）:

```scala
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.11.7")
```

`build.sbt` の対象プロジェクトで `JavaAppPackaging` を有効化する:

```scala
lazy val root = project
  .enablePlugins(JavaAppPackaging)
  .settings(/* ... */)
```

エントリポイント（`Compile / mainClass`）が一意に決まらない場合は明示する。

### 3. project.toml の配置

プロジェクトルートに `project.toml` を作成する。

**sbt 2 系の場合**（タスク名と成果物パスがデフォルトと異なるため必須）:

```toml
[_]
schema-version = "0.2"

[io.buildpacks]
exclude = [
  "target/",
  ".bloop/",
  ".bsp/",
  ".metals/",
]

[[io.buildpacks.build.env]]
name = "BP_SBT_BUILD_ARGUMENTS"
value = "Universal/packageBin"

[[io.buildpacks.build.env]]
name = "BP_SBT_BUILT_ARTIFACT"
value = "target/out/jvm/scala-*/*/universal/*.zip"
```

**sbt 1 系の場合**: buildpack のデフォルト
（`BP_SBT_BUILD_ARGUMENTS=universal:packageBin`、
`BP_SBT_BUILT_ARTIFACT=target/universal/*.zip`）がそのまま使えるので、
`exclude` 指定のみの project.toml でよい:

```toml
[_]
schema-version = "0.2"

[io.buildpacks]
exclude = [
  "target/",
  ".bloop/",
  ".bsp/",
  ".metals/",
]
```

マルチモジュール構成でルート以外のモジュールをビルドする場合は
`BP_SBT_BUILT_MODULE` でモジュール名を指定する。

### 4. ビルド

```bash
pack build <イメージ名> --builder paketobuildpacks/ubuntu-noble-builder
```

- 初回は builder イメージの取得と依存解決が走るため時間がかかる（数分〜）。
  タイムアウトを長め（10分程度）に設定して実行する
- ビルド失敗時は buildpack のログに sbt の出力がそのまま出るので、
  まずローカルで `sbt Universal/packageBin`（sbt 1 なら `sbt universal:packageBin`）が
  通るかを確認する

### 5. 動作確認

```bash
docker run --rm -p 8080:8080 <イメージ名>
```

起動ログに `Calculated JVM Memory Configuration: -Xmx...` の行があれば
メモリ自動設定が機能している。ポート番号はアプリケーションに合わせる。

## オプション設定

必要に応じて `--env` で指定する（`project.toml` の `[[io.buildpacks.build.env]]` でも可）。

| 目的 | 指定 |
|------|------|
| JRE バージョン指定 | `--env BP_JVM_VERSION=21` |
| JDK ではなく JRE を同梱 | `--env BP_JVM_TYPE=JRE` |
| jlink でランタイムを最小化 | `--env BP_JVM_JLINK_ENABLED=true` |
| レジストリへ直接 push | `pack build ghcr.io/<owner>/<name> --publish` |

Amazon Corretto を使う場合（イメージサイズは大きくなりがち）:

```bash
pack build <イメージ名> \
    --builder paketobuildpacks/ubuntu-noble-builder \
    --buildpack docker.io/paketobuildpacks/amazon-corretto \
    --buildpack paketo-buildpacks/java \
    --env BP_JVM_VERSION=21
```

## 注意点

- **UBI 系 builder は避ける**: UBI 8 は glibc の互換性問題で sbt が動作しないことがある。
  `paketobuildpacks/ubuntu-noble-builder` を既定とする
- **静的ファイルの同梱**: `src/universal/` に置いたファイルは
  実行時に `./<プロジェクト名>/` 配下に配置される
- **実行時の JVM オプション**: `docker run -e JAVA_OPTS=...` で環境変数として
  渡すだけで読み込まれる。イメージの再ビルドは不要
- buildpack の詳細設定は https://github.com/paketo-buildpacks/sbt を参照する
