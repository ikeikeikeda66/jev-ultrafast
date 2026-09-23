# Windows 企業環境向け SemIf & Jev Ultrafast 導入・運用手順書 (Claude Code 連携)

本ドキュメントは、企業の Windows 環境（Windows 10 / 11）において、**Claude Code** からローカル推論エンジン **SemIf** および高速自律ブラウザエージェント **jev-ultrafast** を安全かつ確実にセットアップ・運用するための一連の手順書です。

社内セキュリティやプロキシ、管理者権限の制約を考慮した実践的な手順を網羅しています。

---

## 全体アーキテクチャ

```text
┌─────────────────────────────────────────────────────────────┐
│                       Windows PC                            │
│                                                             │
│   ┌────────────────┐      stdio (MCP)      ┌─────────────┐  │
│   │  Claude Code   │ ────────────────────> │   jev_mcp   │  │
│   │   (CLI Tool)   │                       │ (MCP Server)│  │
│   └────────────────┘                       └──────┬──────┘  │
│                                                   │         │
│                                      REST (/v1/decide)      │
│                                                   ▼         │
│   ┌────────────────┐      CDP / DevTools   ┌─────────────┐  │
│   │ Google Chrome  │ <──────────────────── │    SemIf    │  │
│   │ (Remote Debug) │                       │  Resident   │  │
│   └────────────────┘                       │   Server    │  │
│                                            │ (Port 8765) │  │
│                                            └─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. 前提ソフトウェアの準備

Windows の PowerShell または コマンドプロンプト を開いて準備します。
（PowerShell は通常ユーザー権限で動作します）

### ① パッケージマネージャー `uv` のインストール（推奨）
Python 環境の管理には、管理者権限不要で超高速な `uv` を使用します。
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
※ インストール後、PowerShell を再起動するか `$env:Path = [System.Environment]::GetEnvironmentVariable("Path","User")` を実行します。

### ② Python 3.12 の準備
`uv` 経由でシステムを汚さず Python 3.12 をインストールできます：
```powershell
uv python install 3.12
```

### ③ Node.js & Claude Code のインストール
Claude Code は Node.js 製の CLI です。
1. 公式サイト（https://nodejs.org/）から LTS 版（v20 以上）をインストール。
2. Claude Code をグローバルインストール：
```powershell
npm install -g @anthropic-ai/claude-code
```
3. 動作確認：
```powershell
claude --version
```

### ④ Google Chrome の確認
ブラウザ自動化対象として Google Chrome がインストールされていることを確認します。

---

## 2. SemIf サーバーのセットアップ & 起動

SemIf は Apple Silicon だけでなく、Windows の **CPU（AVX2）** または **NVIDIA GPU（CUDA）** でも高速に動作します（llama.cpp / GGUF 版）。

### ① リポジトリの取得
任意の作業フォルダー（例: `C:\AI\semlf`）にクローンします：
```powershell
cd C:\AI
git clone https://github.com/ikeikeikeda66/SemIf.git semlf
cd semlf
```

### ② 依存関係のインストール
```powershell
uv venv --python 3.12
uv sync
```
※ NVIDIA GPU (CUDA) をお持ちの場合は、必要に応じて CUDA 版の llama-cpp-python / PyTorch を指定できます。標準の CPU (Q4_K_M 量子化) でも 4B モデルは数秒で高速に動作します。

### ③ サーバーの起動
Windows 用の起動バッチまたは PowerShell コマンドで起動します：
```powershell
# CPU で起動する場合
uv run python scripts/semif_server.py --host 127.0.0.1 --port 8765 --device auto

# NVIDIA GPU (CUDA) を利用する場合
uv run python scripts/semif_server.py --host 127.0.0.1 --port 8765 --device cuda
```

### ④ ヘルスチェックの確認
別の PowerShell ウィンドウを開き、正常に応答するか確認します：
```powershell
curl.exe -s http://127.0.0.1:8765/health
```
レスポンスで `"ready": true` が返ってくれば準備完了です。

---

## 3. jev-ultrafast のセットアップ

### ① リポジトリの取得
```powershell
cd C:\AI
git clone https://github.com/ikeikeikeda66/jev-ultrafast.git
cd jev-ultrafast
```

### ② 依存関係のインストール
```powershell
uv venv --python 3.12
uv sync
```

### ③ 環境設定ファイル `.env` の作成
`.env.example` をコピーして `.env` を作成します：
```powershell
copy .env.example .env
```
`.env` の内容を確認・編集します：
```ini
# SemIf サーバーのアドレス（デフォルトでローカル 8765）
SEMIF_BASE_URL=http://127.0.0.1:8765

# TYPE_TEXT（入力テキスト生成）用の OpenAI 互換キー
# 社内 Azure OpenAI や DeepSeek、OpenRouter などのキーを設定
TEXT_MODEL_API_KEY=your-api-key-here
TEXT_MODEL_BASE_URL=https://api.deepseek.com/v1
TEXT_MODEL=deepseek-chat
```

### ④ 疎通テストの実行
```powershell
uv run pytest
```
全テストがパスすることを確認します。

---

## 4. Claude Code への MCP 登録

Claude Code から `jev-ultrafast` をツールとして呼び出せるように登録します。

### 方法 A: プロジェクト設定ファイル `.mcp.json` の配置（推奨）
Claude Code を実行する作業ディレクトリ（またはプロジェクトのルート）に `.mcp.json` を作成します：

```json
{
  "mcpServers": {
    "jev-ultrafast": {
      "command": "C:\\AI\\jev-ultrafast\\.venv\\Scripts\\python.exe",
      "args": [
        "C:\\AI\\jev-ultrafast\\scripts\\jev_mcp.py"
      ]
    }
  }
}
```
※ パス内のバックスラッシュ（`\`）は、JSON 内では二重バックスラッシュ（`\\`）でエスケープしてください。

### 方法 B: CLI コマンドによるグローバル登録
```powershell
claude mcp add jev-ultrafast C:\AI\jev-ultrafast\.venv\Scripts\python.exe C:\AI\jev-ultrafast\scripts\jev_mcp.py
```

---

## 5. Claude Code からの利用・実行手順

### ① 事前準備
1. **SemIf サーバー** がバックグラウンドで起動していること（`http://127.0.0.1:8765/health` が `ready: true`）。
2. **Google Chrome** がインストールされていること。

### ② Claude Code の起動
```powershell
cd C:\YourWorkDir
claude
```

### ③ 指示プロンプトの例
Claude Code の対話プロンプトで、ブラウザ操作タスクを依頼します：

> **指示例 1 (航空券検索)**:
> 「`jev_browse` ツールを使って、Google Flights (https://www.google.com/travel/flights?hl=en) で東京（羽田/成田）から福岡への片道航空券を検索して、最安フライト候補を教えて。」

> **指示例 2 (社内ポータルやWeb調査)**:
> 「`jev_browse` ツールで https://example.com を開き、ログインフォームに移動して状態を調べて。」

Claude Code は自動的に `jev_browse` ツールを起動し、SemIf によるミリ秒単位の判断でフォーム入力やクリックを実行し、結果をまとめて返答します。

---

## 6. 社内・企業ネットワーク環境でのトラブルシューティング

### ① 社内プロキシ（Proxy）のバイパス設定
社内プロキシが設定されている場合、ローカルホスト（`127.0.0.1`）への通信がプロキシに吸い込まれて接続エラーになることがあります。
**対処法**: 環境変数 `NO_PROXY` に `127.0.0.1,localhost` を設定します。
```powershell
$env:NO_PROXY = "127.0.0.1,localhost"
[System.Environment]::SetEnvironmentVariable("NO_PROXY", "127.0.0.1,localhost", "User")
```

### ② 社内 SSL インスペクション（自己署名証明書エラー）
Node.js や Python で `CERTIFICATE_VERIFY_FAILED` が発生する場合：
- **npm / Node.js**:
  ```powershell
  $env:NODE_EXTRA_CA_CERTS = "C:\path\to\company_root_ca.crt"
  ```
- **Python (httpx / requests)**:
  ```powershell
  $env:SSL_CERT_FILE = "C:\path\to\company_root_ca.crt"
  $env:REQUESTS_CA_BUNDLE = "C:\path\to\company_root_ca.crt"
  ```

### ③ Chrome のリモートデバッグ権限・ポートの競合
- `browser-harness` が Chrome を起動する際、Chrome の起動オプションでデバッグポート（`--remote-debugging-port=9222` 等）が使用されます。
- すでに起動している既存の Chrome がある場合は、一度 Chrome をすべて閉じるか、独立したユーザーデータディレクトリで起動してください：
  ```powershell
  uv run browser-harness --doctor
  ```

### ④ Windows サービス（常駐タスク）化について
PC 起動時に SemIf サーバーを自動起動したい場合は、Windows の **タスクスケジューラ** を利用して「ログオン時に `run_server.bat` をバックグラウンド実行」するように設定すると便利です。
