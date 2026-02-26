# 同一リポジトリ・複数 fetch:template 実行時のリモートアクセス回数検証

## 目的

**同一リポジトリ**の**異なるパス**（`skeleton-a`, `skeleton-b`, `skeleton-c`）を、**複数回**の `fetch:template` ステップで取得したときに、リモートリポジトリ（Git ホスト）へのアクセスが**何回**発生するかを検証する。

## 検証対象テンプレート

- **ファイル**: `template-multi-fetch.yaml`
- **内容**: 3 つのステップでそれぞれ `fetch:template` を実行
  - Step 1: `url: ./skeleton-a` → `targetPath: ./out-skeleton-a`
  - Step 2: `url: ./skeleton-b` → `targetPath: ./out-skeleton-b`
  - Step 3: `url: ./skeleton-c` → `targetPath: ./out-skeleton-c`

いずれも**同じテンプレートリポジトリ**内の相対パスであり、`baseUrl` はテンプレートの URL に揃う。

## 想定される挙動（実装ベース）

- `fetch:template` は**各ステップごとに** `fetchContents()` を 1 回呼び出す。
- `fetchContents()` は内部で `UrlReaderService#readTree()` を使い、指定 URL（ここでは `baseUrl + fetchUrl`）のツリーを取得する。
- したがって **ステップ数と同数の `readTree` 呼び出し**が発生する可能性が高い（**最大 3 回**のリモートアクセス）。
- 一方で、UrlReader の実装によっては **ETag 等によるキャッシュ**が効き、同一リポジトリ・同一 ref に対して 2 回目以降がキャッシュヒットする場合、**実質のリモートアクセスは 1 回**になる可能性もある。

実際の回数は **Backstage のバージョンと UrlReader の実装**に依存するため、以下で「回数」を計測して検証する。

---

## 検証方法

### 方法 1: Backend ログで readTree / fetch の回数を数える（推奨）

Scaffolder バックエンドで、テンプレート取得や `readTree` に紐づくログが 1 回の fetch につき 1 回出るようにログレベルを上げ、**そのログが何回出たか**を数える。

1. **Backstage のログレベルを変更**
   - `app-config.yaml` や環境変数でログレベルを `debug` にし、Scaffolder または UrlReader 関連のログを有効にする。
   - 例（環境変数）:
     ```bash
     LOG_LEVEL=debug
     ```
   - または、Backstage の `backend` で使っているロガー（例: `winston`）の設定で、`@backstage/plugin-scaffolder-backend` や `UrlReader` を扱うモジュールを `debug` に設定する。

2. **テンプレートを実行**
   - Create から「fetch:template 複数回実行・リモートアクセス回数検証用」テンプレートを選択し、実行する。

3. **ログを確認**
   - 実行中〜完了までの Backend ログから、次のようなメッセージを検索する:
     - `"Fetching template content from remote URL"`（fetch:template の handler 内のログ）
     - または `readTree` / `fetchContents` に紐づくログ（実装によって文言は異なる）
   - **該当ログが 3 回出れば**、少なくとも **3 回**「テンプレート取得のための処理」が走っている（＝リモート参照が 3 回発生している可能性が高い）。
   - **1 回だけ**であれば、何らかのキャッシュ（同一 repo のツリーを再利用等）が効いている可能性がある。

4. **記録するもの**
   - 上記ログの**出現回数**
   - Backstage のバージョン、使用している UrlReader（GitHub / GitLab / 等）

---

### 方法 2: ネットワークレベルでリモートアクセス回数を数える

Git ホスト（GitHub / GitLab 等）への **HTTP(S) リクエストの回数**を計測する。

1. **計測の準備**
   - Backstage バックエンドが動くホストで、次のいずれかを行う:
     - **tcpdump / Wireshark**: 該当ホスト宛の HTTPS トラフィックをキャプチャし、リクエスト数を数える。
     - **プロキシ**（mitmproxy, Charles 等）: バックエンドの HTTP クライアントをプロキシ経由にし、プロキシのアクセスログで「テンプレートリポジトリの URL へのリクエスト」の回数を数える。
   - テンプレートが **GitHub** の場合は、`api.github.com` や `raw.githubusercontent.com` 等へのリクエストが、1 回の `readTree` で複数回になる場合がある（blob 取得など）ため、「**リポジトリのコンテンツ取得に使われたリクエストの塊**」が何回あるか、という観点で数えるとよい。

2. **テンプレートを実行**
   - 方法 1 と同様に、検証用テンプレートを 1 回実行する。

3. **結果の記録**
   - テンプレートリポジトリ用の URL（例: `https://github.com/org/repo` やその API）への **リクエスト回数**、または **readTree に相当しそうなリクエストの塊の回数**を記録する。
   - 3 ステップで 3 回 fetch なら、少なくとも **3 回**の塊（または 3 回以上の HTTP リクエスト）が出る可能性が高い。

---

### 方法 3: ソースコードで呼び出し関係を確認する（補足）

「理論上、何回 readTree が呼ばれるか」を確認する場合:

1. `@backstage/plugin-scaffolder-backend` の `fetch:template` の **handler** では、**1 ステップにつき 1 回** `fetchContents()` が呼ばれる。
2. `fetchContents()` は `UrlReaderService#readTree()` を使って、`baseUrl` と `fetchUrl` を組み合わせた URL のツリーを取得する。
3. したがって、**キャッシュが無い場合**は **fetch:template のステップ数 = readTree の呼び出し回数** となる。

キャッシュの有無は、使用している UrlReader（GitHub, GitLab, Bitbucket 等）の実装や、`readTree` の ETag キャッシュの有無に依存する。検証時は方法 1 または 2 で**実際の回数**を計測することを推奨する。

---

## テンプレートの登録

このリポジトリを Backstage のカタログに登録する。

例（`app-config.yaml`）:

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/<org>/backstage-fetch-template-test/blob/main/template-multi-fetch.yaml
```

または、同一リポジトリに複数テンプレートを登録する場合:

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/<org>/backstage-fetch-template-test/blob/main/template.yaml
    - type: url
      target: https://github.com/<org>/backstage-fetch-template-test/blob/main/template-multi-fetch.yaml
```

---

## 検証結果の記録例

| 項目 | 値 |
|------|-----|
| Backstage バージョン | 例: 1.xx |
| テンプレート | template-multi-fetch.yaml（3 ステップ） |
| 検証方法 | 方法 1: Backend ログ / 方法 2: ネットワーク |
| 観測した「fetch/readTree」回数 | 例: 3 回 |
| リモート（Git ホスト）へのリクエスト回数 | 例: 3 回（または 1 回でキャッシュ） |
| 備考 | 例: LOG_LEVEL=debug で「Fetching template...」をカウント |

この表をコピーし、実際の環境で計測した値で埋めておくと、今後のバージョン比較やキャッシュ挙動の確認に使える。
