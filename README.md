# Google Search Console MCP Server（社内フォーク／日本語版）

[Google Search Console](https://search.google.com/search-console/about)（GSC）をAIアシスタントに接続し、自然言語でSEOデータを分析できるようにするModel Context Protocol（MCP）サーバーです。**Claude Desktop**、**Cursor**、**Codex CLI**、**Gemini CLI**、**Antigravity**、その他MCP対応クライアント全般で利用できます。

本READMEは、本家 [AminForou/mcp-gsc](https://github.com/AminForou/mcp-gsc) をフォークし、日本語ドキュメント化＋社内向けの認証方式（後述）を追加した社内フォークのものです。

---

## なぜこのフォークを作ったか（Why）

本家のGSC MCPサーバーは認証方式として「OAuthブラウザログイン」または「サービスアカウントJSONキー（`GSC_CREDENTIALS_PATH`）」の2つしかサポートしていません。しかし、私たちの実行基盤（AWS Bedrock AgentCore上のコンテナ）では、この2つがどちらも使えませんでした。

- **OAuthブラウザログイン**：サーバーはヘッドレスなコンテナで動くため、ブラウザを開いてログインすることができません。
- **サービスアカウントJSONキー**：長期間有効なキーファイルをコンテナイメージやSecrets Managerに保管する必要があり、キーローテーションの手間や漏洩リスクが増えます。私たちは既にGoogle Analytics / BigQuery MCP連携で **AWS Workload Identity Federation（WIF）** を採用しており、長期キーを一切持たない運用にしています。GSCだけ別方式（サービスアカウントキー）にするのは一貫性がなく、セキュリティ的にも後退でした。

### 具体的な使い方（WIFでの活用方法）

1. AWSの実行ロール（例: Lambda/FargateなどのIAMロール）が持つAWS STSの一時クレデンシャル（`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`）を取得し、MCPサーバーを起動する子プロセスの環境変数として渡す。
2. GCP側のWorkload Identity Federation設定ファイル（`type: external_account`、AWSのIAMロールをGCPサービスアカウントに偽装（impersonate）させる設定）を用意する。EC2以外（Lambda/Fargateなど）で動かす場合は、設定ファイル内の`credential_source.region_url`/`credential_source.url`（EC2メタデータサーバー参照用）を削除し、代わりに`environment_id: "aws1"`方式（環境変数経由でAWS一時クレデンシャルを渡す方式）に調整する。
3. その設定ファイルのパスを環境変数`GOOGLE_APPLICATION_CREDENTIALS`としてMCPサーバーに渡す。
4. 本フォークで追加した`get_gsc_service()`のADC（Application Default Credentials）フォールバックが、`google.auth.default(scopes=SCOPES)`経由でこのWIF設定を自動的に読み込み、ブラウザ操作や長期キーなしでSearch Console APIを呼び出せるようにする。

この変更（`get_gsc_service()`へのADCフォールバック追加）は本家へのIssue/PR提案も検討中ですが、まずは社内フォークとして先行運用しています。詳細は `gsc_server.py` の `get_gsc_service()` 内のコメント、および環境変数リファレンスの `GOOGLE_APPLICATION_CREDENTIALS` / `GSC_USE_ADC` の項目を参照してください。

言葉だけだとイメージしづらいので、[Strands Agents](https://strandsagents.com/)のMCPクライアントからこのフォークを呼び出すサンプルコードを載せます（AWS STSの一時クレデンシャルをそのまま環境変数として子プロセスに渡すだけのシンプルな例です）。

```python
import asyncio
import os
import json
import boto3

from mcp import StdioServerParameters, stdio_client
from strands.tools.mcp import MCPClient

async def gsc_sample():

    # GCPコンソール > IAMと管理 > Workload Identity プール > 対象のプロバイダー
    # > 「構成をダウンロード」から取得したJSONファイルのパス。
    # EC2ではなくFargate/LambdaなどでAWS STSクレデンシャルを
    # 環境変数経由で渡す場合は、ダウンロードしたJSON内の
    # credential_source.region_url / credential_source.url を削除しておくこと。
    wi_json_path = "/path/to/workload-identity-config.json"

    session = boto3.Session()
    creds = session.get_credentials()
    frozen_creds = creds.get_frozen_credentials()

    gsc = MCPClient(
        lambda: stdio_client(
            StdioServerParameters(
                command="uvx",
                # 本家PyPI版ではなく、このADC対応フォークを明示的に指定する
                args=[
                    "--from", "git+https://github.com/okamoto-dts-stg-com/mcp-gsc@v0.3.2-adc2",
                    "mcp-search-console",
                ],
                env={
                    "GOOGLE_APPLICATION_CREDENTIALS": wi_json_path,
                    "GSC_SKIP_OAUTH": "true",  # ヘッドレス環境なのでOAuthブラウザフローをスキップ
                    "AWS_ACCESS_KEY_ID": frozen_creds.access_key,
                    "AWS_SECRET_ACCESS_KEY": frozen_creds.secret_key,
                    "AWS_SESSION_TOKEN": frozen_creds.token,
                    "AWS_REGION": session.region_name,
                },
            )
        )
    )

    with gsc:
        tools = gsc.list_tools_sync()
        result = await gsc.call_tool_async(
            tool_use_id="test-1",
            name="list_properties",
            arguments={}
        )
        print(json.dumps(result, indent=2, ensure_ascii=False))

if __name__ == "__main__":
    asyncio.run(gsc_sample())
```

---

## 更新履歴（このフォーク独自の変更）

### 社内パッチ — 2026年7月
- **ADC（Application Default Credentials）フォールバックを追加** — `GOOGLE_APPLICATION_CREDENTIALS`が設定されている場合（または`GSC_USE_ADC=true`の場合）、`google.auth.default()`経由で認証を試みるようにした。Workload Identity Federation・GCE/Cloud Runメタデータ・gcloudユーザーADCなど、サービスアカウントJSONキー以外の資格情報でも動作するようになる。既存のOAuth／サービスアカウントJSONキーの挙動には一切変更なし（追加のフォールバックのみ）。

### 本家の更新履歴

#### [0.3.2] — 2026年4月
- **uvx利用時のOAuthブラウザフローを修正** — macOS上でMCPサブプロセスとして実行した際にブラウザログイン画面が開かない原因だった`isatty`チェックを削除。`uvx`だけでOAuthがそのまま動作するようになった。
- **`get_capabilities`ツールを追加** — 一度の呼び出しで利用可能な全ツール一覧と現在の認証状態を確認できる。AIアシスタントがどのツールを使えるか分からない場合に有用。
- **認証エラーメッセージを改善** — 資格情報が不足・失効している場合に、具体的な対処方法を全ツールが案内するようになった。

---

## できること

**プロパティ管理**
- 保有する全GSCプロパティを一覧表示
- 所有権確認の詳細情報を取得
- アカウントへのプロパティ追加・削除

**検索アナリティクス・レポート**
- サイトに流入している検索クエリを把握
- 表示回数・クリック数・CTRを追跡
- パフォーマンス推移の分析、期間比較
- AIアシスタントが作成するグラフでデータを可視化

**URL検査・インデックス状況**
- 特定ページのインデックス問題を確認
- Googleが最後にクロールした日時を確認
- 複数URLを一括検査してパターンを把握

**サイトマップ管理**
- 全サイトマップとその状態を確認
- 新規サイトマップの送信
- エラー・警告の確認

---

## 利用可能なツール

| ツール | 内容 | 必要な入力 |
|------|-------------|--------------------------|
| `get_capabilities` | 全ツール一覧と認証状態を表示。迷ったらまずこれを呼ぶ | なし |
| `list_properties` | 全GSCプロパティを表示 | なし |
| `get_site_details` | 特定サイトの詳細情報 | Site URL |
| `get_search_analytics` | クリック・表示回数・CTR・順位を含む上位クエリ・ページ | Site URL, 期間 |
| `get_performance_overview` | サイトパフォーマンスのサマリ | Site URL, 期間 |
| `compare_search_periods` | 2つの期間のパフォーマンス比較 | Site URL, 2つの日付範囲 |
| `get_search_by_page_query` | 特定ページへの流入クエリ | Site URL, page URL |
| `get_advanced_search_analytics` | 国・デバイス・クエリ・ページで絞り込む高度なアナリティクス | Site URL |
| `inspect_url_enhanced` | URLのクロール・インデックス状況の詳細 | Site URL, page URL |
| `batch_url_inspection` | 最大10件のURLを一括検査 | Site URL, URLリスト |
| `check_indexing_issues` | 複数URLのインデックス問題を確認 | Site URL, URLリスト |
| `get_sitemaps` | サイトの全サイトマップ一覧 | Site URL |
| `list_sitemaps_enhanced` | エラー・警告を含むサイトマップの詳細 | Site URL |
| `manage_sitemaps` | サイトマップの送信・削除 | Site URL, action |
| `reauthenticate` | OAuthログインをやり直す（アカウント切り替え） | なし |

*全20ツールの一覧は、AIアシスタントに「get_capabilitiesを呼んで」と依頼すれば確認できます。*

---

## はじめに

### 手順1 — Google API認証情報の設定

どのクライアントで使う場合も、事前に認証情報が必要です。以下のいずれかを選んでください。

#### オプションA — OAuth（推奨　自分のGoogleアカウントを使用）

1. [Google Cloud Console](https://console.cloud.google.com/) でプロジェクトを作成／選択
2. [Search Console APIを有効化](https://console.cloud.google.com/apis/library/searchconsole.googleapis.com)
3. [認証情報](https://console.cloud.google.com/apis/credentials) → 認証情報を作成 → **OAuthクライアントID**
4. OAuth同意画面を設定し、**デスクトップアプリ**を選択して作成
5. JSONファイルをダウンロードし、永続的な場所に保存（例: `~/Documents/client_secrets.json`）

初回利用時にブラウザが開き、Googleアカウントログインを求められます。以後はトークンが保存され、ブラウザ操作は不要になります。

#### オプションB — サービスアカウント（自動化・チーム利用向け）

1. [Google Cloud Console](https://console.cloud.google.com/) でプロジェクトを作成／選択
2. [Search Console APIを有効化](https://console.cloud.google.com/apis/library/searchconsole.googleapis.com)
3. [認証情報](https://console.cloud.google.com/apis/credentials) → 認証情報を作成 → **サービスアカウント**
4. 「キー」タブ → キーを追加 → 新しいキーを作成 → JSON → ダウンロード
5. 永続的な場所に保存（例: `~/Documents/service_account.json`）
6. そのサービスアカウントのメールアドレスをGSCプロパティに登録：Search Console → 設定 → ユーザーと権限 → ユーザーを追加 → フル権限

#### オプションC — Workload Identity Federation / ADC（このフォークで新規追加、ヘッドレスサーバー向け）

サービスアカウントJSONキーをコンテナに配置したくない場合（AWS Fargateなど）の方法です。

1. GCP側でWorkload Identity PoolとProviderを作成し、AWSのIAMロールがGCPサービスアカウントを偽装（impersonate）できるように設定する（GCP公式ドキュメント参照）。
2. そのサービスアカウントのメールアドレスをGSCプロパティの「ユーザーと権限」に登録（Option Bの手順6と同じ）。
3. GCPが発行するWIF設定JSON（`type: external_account`）を取得し、AWS EC2メタデータ用の`credential_source.region_url`/`credential_source.url`を削除して代わりに`environment_id: "aws1"`方式（環境変数経由でAWS一時クレデンシャルを渡す方式）に調整する。
4. MCPサーバー起動時に以下の環境変数を渡す：
   - `GOOGLE_APPLICATION_CREDENTIALS`: 上記WIF設定JSONのパス
   - `GSC_SKIP_OAUTH=true`（ブラウザフローをスキップ）
   - `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` / `AWS_REGION`（AWS一時クレデンシャル。`environment_id: "aws1"`方式はAWS_PROFILEではなくこれらの環境変数を直接参照する）

内部的には`get_gsc_service()`が`google.auth.default(scopes=SCOPES)`を呼び、`GOOGLE_APPLICATION_CREDENTIALS`の内容（`external_account`型を含む）を自動判別して認証します。具体的なコード例は上の「[なぜこのフォークを作ったか（Why）](#なぜこのフォークを作ったかwhy)」内のサンプルコードを参照してください。

### 手順2 — インストール

#### オプションA — uvx（推奨）

クローン不要、Pythonインストール不要、仮想環境不要。`uvx`がサーバーを自動ダウンロード・実行し、常に最新に保ちます。

**uvのインストール** — ターミナルを開き、以下3つを順番に実行：

```bash
# 1. ダウンロードとインストール
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. 現在のターミナルセッションで有効化
source $HOME/.local/bin/env

# 3. 今後の全セッションで永続化
echo 'source $HOME/.local/bin/env' >> ~/.zshrc
```

確認：
```bash
uv --version
```

> **なぜ3つ必要か？** インストーラーは`uv`を`~/.local/bin`に配置しますが、既に開いているターミナルセッションはそのフォルダをまだ認識していません。手順2で即座に有効化し、手順3で以降の全ターミナルウィンドウで自動反映させます。

AIクライアントを設定：

---

**Claude Desktop**

設定ファイル: `~/Library/Application Support/Claude/claude_desktop_config.json`

OAuth:
```json
{
  "mcpServers": {
    "gscServer": {
      "command": "/FULL/PATH/TO/uvx",
      "args": ["mcp-search-console"],
      "env": {
        "GSC_OAUTH_CLIENT_SECRETS_FILE": "/full/path/to/client_secrets.json"
      }
    }
  }
}
```

サービスアカウント:
```json
{
  "mcpServers": {
    "gscServer": {
      "command": "/FULL/PATH/TO/uvx",
      "args": ["mcp-search-console"],
      "env": {
        "GSC_CREDENTIALS_PATH": "/full/path/to/service_account.json",
        "GSC_SKIP_OAUTH": "true"
      }
    }
  }
}
```

---

**Cursor**

設定ファイル: `~/.cursor/mcp.json`

OAuth:
```json
{
  "mcpServers": {
    "gscServer": {
      "command": "/FULL/PATH/TO/uvx",
      "args": ["mcp-search-console"],
      "env": {
        "GSC_OAUTH_CLIENT_SECRETS_FILE": "/full/path/to/client_secrets.json"
      }
    }
  }
}
```

---

**Codex CLI**

設定ファイル: `~/.codex/config.toml`

OAuth:
```toml
[mcp_servers.gscServer]
command = "/FULL/PATH/TO/uvx"
args = ["mcp-search-console"]
enabled = true
env = { GSC_OAUTH_CLIENT_SECRETS_FILE = "/full/path/to/client_secrets.json" }
```

サービスアカウント:
```toml
[mcp_servers.gscServer]
command = "/FULL/PATH/TO/uvx"
args = ["mcp-search-console"]
enabled = true
env = { GSC_CREDENTIALS_PATH = "/full/path/to/service_account.json", GSC_SKIP_OAUTH = "true" }
```

---

> **uvxのパスの見つけ方：** macOS/Linuxでは、uvインストール後に`which uvx`を実行（通常は`/Users/YOUR_NAME/.local/bin/uvx`）。Windowsでは、PowerShellで`Get-Command uvx | Select-Object -ExpandProperty Source`（またはcmdで`where uvx`）を実行（通常は`C:\Users\YOUR_NAME\.local\bin\uvx.exe`）。上記設定の`/FULL/PATH/TO/uvx`をそのパスに置き換えてください。
>
> **なぜフルパスが必要か？** Claude DesktopやCursorなどのGUIアプリはシェル設定（`~/.zshrc`）を読まずに起動するため、`~/.local/bin`を認識できません。フルパスを使えば、起動方法に依存せず確実に動作します。`spawn uvx ENOENT`エラーが出たらこれが原因です。

設定保存後、**アプリを完全に終了（`Cmd+Q`）して再起動**してください。

OAuthの場合：初回利用時にブラウザが自動で開きログインを求められます。以後はトークンがキャッシュされ、再度求められません。

---

#### オプションB — クローン（上級者向け）

コードを修正したい場合や、特定のローカルバージョンを実行したい場合はこちらを使用してください。

> **Python 3.11+が必須です。** Python 3.10以下ではサーバーが起動しません。Claude DesktopなどGUIクライアントから起動された場合、ツールが1つも表示されずログも出ないため、原因特定が困難です。`python --version`でバージョンを確認し、3.11未満ならアップデートしてください。uvx方式（オプションA）ならPythonバージョンを自動管理するため、この問題を回避できます。

**リポジトリをクローン：**
```bash
git clone https://github.com/okamoto-dts-stg-com/mcp-gsc.git
cd mcp-gsc
```

**環境をセットアップ：**
```bash
uv venv .venv
uv pip install -r requirements.txt
```

**AIクライアントを設定**（Claude Desktopの例）：

OAuth:
```json
{
  "mcpServers": {
    "gscServer": {
      "command": "/full/path/to/mcp-gsc/.venv/bin/python",
      "args": ["/full/path/to/mcp-gsc/gsc_server.py"],
      "env": {
        "GSC_OAUTH_CLIENT_SECRETS_FILE": "/full/path/to/client_secrets.json"
      }
    }
  }
}
```

サービスアカウント:
```json
{
  "mcpServers": {
    "gscServer": {
      "command": "/full/path/to/mcp-gsc/.venv/bin/python",
      "args": ["/full/path/to/mcp-gsc/gsc_server.py"],
      "env": {
        "GSC_CREDENTIALS_PATH": "/full/path/to/service_account.json",
        "GSC_SKIP_OAUTH": "true"
      }
    }
  }
}
```

Macパス例：
- Python: `/Users/yourname/Documents/mcp-gsc/.venv/bin/python`
- スクリプト: `/Users/yourname/Documents/mcp-gsc/gsc_server.py`

---

### 手順3 — テスト

AIアシスタントに尋ねてみましょう：**「GSCのプロパティ一覧を見せて」**

プロパティが表示されれば成功です。表示されなければ、**「get_capabilitiesを呼んで」**と依頼して認証状態と問題を診断してください。

---

## 環境変数リファレンス

| 変数 | 必須か | デフォルト | 説明 |
|---|---|---|---|
| `GSC_OAUTH_CLIENT_SECRETS_FILE` | OAuthのみ | — | OAuthクライアントシークレットJSONへの絶対パス。`uvx`利用時は必須。 |
| `GSC_CREDENTIALS_PATH` | サービスアカウントのみ | — | サービスアカウントJSONキーへの絶対パス。`uvx`利用時は必須。`type: service_account`形式のJSONのみ対応。 |
| `GOOGLE_APPLICATION_CREDENTIALS` | ADCのみ（このフォークで新規） | — | ADC経由で認証する場合の資格情報JSONへの絶対パス。Workload Identity Federation（`external_account`）やGCE/Cloud Runメタデータなど、`service_account`以外の形式も自動判別。 |
| `GSC_USE_ADC` | いいえ（このフォークで新規） | `false` | `"true"`にすると、`GOOGLE_APPLICATION_CREDENTIALS`未設定でもADCパスを強制的に試行（GCE/Cloud Runメタデータなどのアンビエント資格情報向け）。 |
| `GSC_SKIP_OAUTH` | いいえ | `false` | `"true"`にするとOAuthを完全にスキップしてサービスアカウント/ADC認証を強制 |
| `GSC_DATA_STATE` | いいえ | `"all"` | `"all"`はGSCダッシュボードと一致。`"final"`は確定済みデータのみ（2～3日の遅延）。 |
| `GSC_ALLOW_DESTRUCTIVE` | いいえ | `false` | `"true"`にするとサイトの追加/削除・サイトマップ削除ツールを有効化 |

---

## Cursor Marketplace

（上流パッケージ`mcp-search-console`はCursor Marketplaceでワンクリックインストールできますが、**この社内フォーク自体はMarketplaceには公開していません**。上流版を使う場合の参考情報として残しています。）

インストール後、認証情報（上記手順1）を設定すれば、Cursor Agentチャットで以下のスキルを利用できます：

| スキル | 呼び出し方 | 内容 |
|---|---|---|
| `seo-weekly-report` | 「example.comのSEO週次レポートを実行して」 | 期間比較と上位クエリを含む週次28日間パフォーマンスサマリ |
| `cannibalization-check` | 「example.comのキーワードカニバライゼーションをチェックして」 | 複数ページが竞合するクエリを検出し、残すべきページを提案 |
| `indexing-audit` | 「上位ページのインデックススを監査して」 | 上位20ページを一括検査し、優先度付き修正リストを返す |
| `content-opportunities` | 「example.comのコンテンツ改善機会を見つけて」 | 表示回数多・CTR低の順位11-20位クエリを抽出 |

---

## サンプルプロンプト

| ツール | サンプルプロンプト |
|------|--------------|
| `list_properties` | 「自分のGSCプロパティを全部表示して、どれが一番インデックスされているか教えて」 |
| `get_search_analytics` | 「mywebsite.comの直近30日間の上位20クエリを見せて、CTRが2%未満のものをハイライトしてタイトル改善案を提案して」 |
| `get_performance_overview` | 「mywebsite.comの直近28日間のパフォーマンス概要をグラフ化し、異常な低下/急増を指摘して原因を推測して」 |
| `check_indexing_issues` | 「このページのインデックス問題を確認して: mywebsite.com/product, mywebsite.com/services, mywebsite.com/about」 |
| `inspect_url_enhanced` | 「mywebsite.com/landing-pageを徹底的に検査して、改善提案をして」 |
| `compare_search_periods` | 「1月と2月のサイトパフォーマンスを比較して、改善したクエリを教えて」 |
| `get_advanced_search_analytics` | 「表示回数が多く順位10以下のクエリを、米国のモバイルトラフィックに絞って分析して」 |

---

## トラブルシューティング

### `spawn uvx ENOENT` または `command not found: uvx`

AIクライアントが`uvx`を見つけられていません。`uvx`だけでなくフルパスを使用してください：

```bash
# フルパスを確認（macOS/Linux）:
which uvx
# 通常: /Users/YOUR_NAME/.local/bin/uvx
```

```powershell
# フルパスを確認（Windows PowerShell）:
Get-Command uvx | Select-Object -ExpandProperty Source
# 通常: C:\Users\YOUR_NAME\.local\bin\uvx.exe
```

設定の`"command": "uvx"`をフルパス（例: `"command": "/Users/YOUR_NAME/.local/bin/uvx"`）に置き換えてください。

### インストール直後に`uv --version`が「command not found」になる

インストーラーは`~/.local/bin`を更新しますが、現在のターミナルセッションはまだ認識していません。以下を実行：

```bash
source $HOME/.local/bin/env
```

永続化するには：
```bash
echo 'source $HOME/.local/bin/env' >> ~/.zshrc
```

### 認証失敗／資格情報ファイルが見つからない

資格情報ファイルは**絶対パス**を使用しているか確認してください（相対パスや`~/`は不可）。例：
```
/Users/yourname/Documents/client_secrets.json   ✅
~/Documents/client_secrets.json                 ✅
client_secrets.json                              ❌
```

### MCPがClaude Desktopアプリでのみ動作し、Webでは動かない

MCPサーバーはローカルで実行されます。[claude.ai/download](https://claude.ai/download)からダウンロードした**Claude Desktopアプリ**でのみ動作し、claude.aiのブラウザ版では動作しません。

### AIクライアントの設定問題

1. 設定内の全ファイルパスが正しい絶対パスか確認
2. 設定変更後は必ずアプリを完全に終了（`Cmd+Q`）して再起動—ウィンドウを閉じるだけでは不十分
3. AIアシスタントに「get_capabilitiesを呼んで」と依頼すれば、正確な認証状態とエラーを報告してくれます

---

## 安全性：破壊的操作

デフォルトで`add_site`、`delete_site`、`delete_sitemap`は無効化されています。有効化するには：

```json
"GSC_ALLOW_DESTRUCTIVE": "true"
```

---

## リモートデプロイ／Docker（上級者向け）

通常はローカルで実行します。このセクションはリモートサーバーやコンテナで実行したい場合のみ対象です。

### HTTPトランスポート

```bash
MCP_TRANSPORT=sse MCP_HOST=0.0.0.0 MCP_PORT=3001 python gsc_server.py
```

| 変数 | デフォルト | 説明 |
|---|---|---|
| `MCP_TRANSPORT` | `stdio` | ネットワーク/リモート利用の場合は`sse`に設定 |
| `MCP_HOST` | `127.0.0.1` | バインドするホスト |
| `MCP_PORT` | `3001` | バインドするポート |

### Docker

```bash
docker build -t mcp-gsc .

docker run \
  -e MCP_TRANSPORT=sse \
  -e MCP_HOST=0.0.0.0 \
  -e MCP_PORT=3001 \
  -e GSC_CREDENTIALS_PATH=/app/credentials.json \
  -v /path/to/credentials.json:/app/credentials.json \
  -p 3001:3001 \
  mcp-gsc
```

（AWS FargateでWIF/ADCを使う場合は、`GSC_CREDENTIALS_PATH`の代わりに`GOOGLE_APPLICATION_CREDENTIALS`をマウントし、AWS一時クレデンシャルを環境変数で注入してください。詳細は本 README 冒頭の「なぜこのフォークを作ったか」を参照。）

---

## 関連ツール

本家作者による関連ツールの情報は [本家リポジトリ](https://github.com/AminForou/mcp-gsc#related-tools) を参照してください。

---

## コントリビュート

このフォークへの修正はIssueまたはPull Requestでどうぞ。WIF/ADC以外の汎用的な修正は、当社内だけではなく[本家](https://github.com/AminForou/mcp-gsc/issues)への逆輸入（upstream）も検討してください。

---

## ライセンス

MITライセンス。詳細は[LICENSE](LICENSE)ファイルを参照。

---

## 変更履歴（本家分）

### [0.3.2] — 2026年4月
- **uvx利用時のOAuthブラウザフローを修正** — macOS上でMCPサブプロセスとして実行した際にブラウザログイン画面が開かない原因だった`isatty`チェックを削除。
- **`get_capabilities`ツール** — 全ツールをカテゴリ別に返し、現在の認証状態も一度に返す。
- **認証エラーメッセージを改善** — 資格情報が不足/失効している場合、`reauthenticate`を呼ぶよう明示する。
- **`list_properties`の説明を改善** — 遅延ツール読み込みを使うクライアントでのツール発見精度を向上。

### [0.3.1] — 2026年4月
- `list_properties`が本当の認証エラーを隠蔽していた不具合を修正。資格情報未設定時は即座に失敗するように。

### [0.3.0] — 2026年4月
- Cursor Marketplaceプラグイン（4つのSEOスキル同梱）
- トークン保存先をプラットフォームのユーザー設定ディレクトリに安定化（`uvx`アップデートでも消えない）
- 全データツールで構造化JSON出力
- 単体テスト39件

### [0.2.2] — 2026年4月
- 破壊的ツールの安全モード（デフォルト無効）
- リモートデプロイ向けHTTP/SSEトランスポート
- Dockerfile

### [0.2.1] — 2026年3月
- アカウント切り替え用`reauthenticate`ツール
- サイトマップのTypeErrorクラッシュを修正
- ドメインプロパティの404エラーを修正

### [0.2.0] — 2026年3月
- `dataState: "all"`をデフォルト化（GSCダッシュボードと一致）
- 柔軟な`row_limit`パラメータ（最大500）
- 高度なアナリティクス向け多次元フィルタリング

### [0.1.0] — 初回リリース
- プロパティ管理・検索アナリティクス・URL検査・サイトマップ管理を網羅する19ツール
- OAuthおよびサービスアカウント認証
