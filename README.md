# Backstage fetch:template 挙動確認用

Backstage の **fetch:template** Action の挙動を確認するための Software Template とスケルトンです。

## 構成

```
.
├── README.md           # 本ファイル
├── template.yaml       # Software Template 定義（Create で選択するテンプレート）
└── skeleton/           # fetch:template が取得・展開するスケルトン
    ├── README.md
    ├── package.json
    ├── catalog-info.yaml
    ├── optional-feature.txt   # addOptionalFile が true のとき内容が変わる
    └── src/
        └── index.ts
```

- **template.yaml**: Create 画面で選ぶテンプレート。`fetch:template` ステップで `./skeleton` を取得し、パラメータを `values` として渡します。
- **skeleton/**: Nunjucks で記述したスケルトン。`{{ name }}` や `{{ description }}` などが実行時に置換されます。

## Backstage での登録方法

1. このリポジトリを Backstage が参照できる場所に置く（Git リポジトリなど）。
2. `app-config.yaml` の `catalog.locations` にテンプレートの URL を追加する。

例（同一リポジトリ内に template と skeleton がある場合）:

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/your-org/backstage-fetch-template-test/blob/main/template.yaml
```

または、`template.yaml` があるディレクトリを指す URL を指定します。

3. Backstage の **Create** から「fetch:template 挙動確認用テンプレート」を選択し、フォームに入力して実行する。

## 挙動確認のポイント

1. **変数置換**
   - コンポーネント名・説明・オーナーを入力して実行する。
   - 生成された `README.md` および `catalog-info.yaml` 内の `{{ name }}` / `{{ description }}` / `{{ owner }}` が、入力値に置換されていることを確認する。

2. **package.json**
   - `name` が `{{ name | lower | replace(' ', '-') }}` により、小文字・スペースはハイフンに変換された値になっていることを確認する。

3. **条件分岐（Nunjucks）**
   - 「オプションを追加する」を **オン** にして実行し、`optional-feature.txt` に「このファイルは…」のメッセージが含まれることを確認する。
   - **オフ** のまま実行し、同じファイルに「addOptionalFile が false のため…」と出ることを確認する。

4. **targetPath（任意）**
   - `template.yaml` の `fetch:template` の `input` に `targetPath: ./my-subdir` を追加して実行し、スケルトンが `my-subdir` 以下に展開されることを確認する。

## 参考

- [Writing Templates](https://backstage.io/docs/features/software-templates/writing-templates)
- [Built-in actions (fetch:template)](https://backstage.io/docs/features/software-templates/builtin-actions#fetch-template)
- fetch:template は Nunjucks でファイル名・内容をレンダリングし、指定 URL（ここでは `./skeleton`）のディレクトリツリーをワークスペースに配置します。
