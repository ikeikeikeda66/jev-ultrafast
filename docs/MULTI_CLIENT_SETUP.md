# Jev Ultrafast マルチクライアント連携ガイド（Claude / Codex / Antigravity）

`jev-ultrafast` は、ローカル常駐の **SemIf**（意味決定モデル）と Chrome ブラウザ操作を組み合わせた超高速自律ブラウザエージェントです。
Model Context Protocol (MCP) を介して、主要な AI エージェント環境から直接ツールとして呼び出して活用できます。

---

## 1. 提供される MCP ツール

| ツール名 | 説明 | 主要引数 |
|---|---|---|
| `jev_browse` | 指定した URL をブラウザで開き、自然言語のゴールを自律的に達成するまで操作を実行 | `url` (必須), `goal` (必須), `max_steps` (任意, デフォルト 10) |
| `jev_decide` | 観測したページ状態とゴールから、SemIf を用いて次に実行すべきアクション（クリック/入力等）を判定 | `page` (必須), `goal` (必須), `history` (任意) |

---

## 2. 各クライアントへの設定状況

### ① Claude Desktop（クロウ）
設定ファイル `~/Library/Application Support/Claude/claude_desktop_config.json` に登録済みです：
```json
{
  "mcpServers": {
    "jev-ultrafast": {
      "command": "/Volumes/SSD_USB_1/AntiGravitiRoot/jev-ultrafast/.venv/bin/python",
      "args": [
        "/Volumes/SSD_USB_1/AntiGravitiRoot/jev-ultrafast/scripts/jev_mcp.py"
      ]
    }
  }
}
```
**利用方法**: Claude Desktop を再起動すると、チャット内で `jev_browse` ツールが有効になります。
> 例: 「Google Flights（https://www.google.com/travel/flights?hl=en）でロンドンからチューリッヒへの9月20日の片道航空券を検索して」

---

### ② Claude Code
プロジェクト直下の `.mcp.json` または CLI コマンドで追加できます：
```bash
claude mcp add jev-ultrafast /Volumes/SSD_USB_1/AntiGravitiRoot/jev-ultrafast/.venv/bin/python /Volumes/SSD_USB_1/AntiGravitiRoot/jev-ultrafast/scripts/jev_mcp.py
```

---

### ③ Codex CLI
グローバル MCP サーバーとして登録済みです（`codex mcp list` で確認可能）：
```bash
# 登録コマンド（実行済み）
codex mcp add jev-ultrafast -- /Volumes/SSD_USB_1/AntiGravitiRoot/jev-ultrafast/.venv/bin/python /Volumes/SSD_USB_1/AntiGravitiRoot/jev-ultrafast/scripts/jev_mcp.py
```
**利用方法**: `codex` コマンドでタスクを実行する際、Codex が必要に応じて自動的に `jev_browse` ツールを呼び出します。

---

### ④ Antigravity (Google DeepMind Antigravity IDE / CLI)
- **MCP 設定**: `~/.gemini/config/mcp_config.json` に `jev-ultrafast` として登録済みです。
- **エージェントスキル**: `~/.gemini/config/skills/jev-ultrafast/SKILL.md` にスキル定義を配備済みです。
Antigravity はブラウザ自動化タスクや航空券・Web フォーム操作のリクエストを受けた際、このスキルおよび MCP ツールを自律的に認識して活用します。

---

## 3. 前提条件（SemIf サーバー）
ブラウザ操作の判断には、ローカル常駐の SemIf サーバー（`http://127.0.0.1:8765`）が起動している必要があります：
```bash
# ヘルスチェック
curl -s http://127.0.0.1:8765/health
```
