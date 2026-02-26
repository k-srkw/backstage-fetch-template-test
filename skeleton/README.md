# {{ name }}

{{ description }}

## 概要

このプロジェクトは Backstage の **fetch:template** Action でスキャフォールドされたスケルトンです。

- **コンポーネント名**: {{ name }}
- **オーナー**: {{ owner }}
- **生成日時**: テンプレート実行時に自動設定されます

## セットアップ

```bash
npm install
```

## 開発

```bash
npm run dev
```

## 確認事項（fetch:template の挙動確認用）

- 上記の `{{ name }}` および `{{ description }}` がフォーム入力値に置換されていること
- `catalog-info.yaml` のメタデータが正しく埋め込まれていること
- `package.json` の `name` が `{{ name }}` から置換されていること
