# サブエージェント: Planner → Generator → Evaluator

短いアイデアから、仕様化 → 実装 → テストを反復で回す3エージェント構成です。
Claude Code は `.claude/agents/*.md` のフロントマター `name` でエージェントを認識します。

## パイプライン

```
 1〜4行のアイデア
        │
        ▼
 ┌──────────────┐   SPEC.md      ┌──────────────┐  自己評価+成果物  ┌──────────────┐
 │  planner     │ ─────────────▶ │  generator   │ ───────────────▶ │  evaluator   │
 │ 何を作るか    │   (機能/SP/    │ 1スプリント   │                  │ Playwrightで  │
 │ に集中        │    受入基準)    │ ずつ実装      │ ◀─────────────── │ 操作してテスト │
 └──────────────┘                └──────────────┘   具体的フィードバック └──────────────┘
                                        ▲                                    │
                                        └──────── 不合格なら修正イテレーション ──┘
```

## 各エージェント

| エージェント | 役割 | 守るべき境界 |
|---|---|---|
| **planner** | 短いプロンプトを詳細仕様書（`SPEC.md`）へ展開。機能一覧・スプリント計画・受け入れ基準・評価閾値を生成。 | **WHAT のみ**。技術的な実装方法（DBスキーマ等）は書かない。誤った実装指定の下流伝播を防ぐため。 |
| **generator** | `SPEC.md` をスプリント方式で1機能ずつ実装。技術選定は裁量。スプリント終了時に自己評価。 | スコープは1スプリント。受け入れ基準を満たすまで完成と呼ばない。 |
| **evaluator** | Playwright MCP で実アプリを操作し、UI/API/永続化を検証。閾値で合否判定し、具体的バグ・改善点を返す。 | 閾値が1つでも割れたら不合格。観測事実のみで判定。 |

## 使い方の例

1. メインスレッドで planner を起動: 「2Dレトロゲームメーカーを作って」→ `SPEC.md` 生成
2. generator を起動: 「SPEC.md のスプリント1を実装して」
3. evaluator を起動: 「スプリント1を評価して」→ PASS/FAIL とフィードバック
4. FAIL なら generator にフィードバックを渡して修正 → 再度 evaluator。PASS したら次スプリントへ。

## 前提: Playwright MCP のセットアップ

**evaluator** は Playwright MCP（`mcp__playwright`）で実ブラウザを操作します。本リポジトリには
プロジェクト共有の MCP 設定 `.mcp.json` を同梱済みです（`npx @playwright/mcp` を `--headless --isolated` で起動）。

新しいセッションで `.mcp.json` を初めて読み込む際は、信頼確認のプロンプトが出ます。許可すると
`playwright` サーバーが有効になり、evaluator が `mcp__playwright__*` ツールを使えるようになります。

### ブラウザ本体のインストール（初回のみ）

Playwright はブラウザ本体を別途ダウンロードします。実行環境で一度だけ：

```bash
npx playwright install chromium
# OS依存ライブラリも入れる場合（apt が使える環境）:
# npx playwright install --with-deps chromium
```

### ネットワークポリシーに関する注意（Claude Code on the web / リモート環境）

リモート実行環境では送信先がネットワークポリシーで制限されます。ブラウザは
`cdn.playwright.dev` から配信されるため、ここが許可リストにないと
`403 Host not in allowlist` でダウンロードに失敗します。その場合は次のいずれか：

1. **ローカル / デスクトップアプリ**で evaluator を実行する（ブラウザ取得が可能）。
2. リモート環境の**ネットワークポリシーを緩める**（`cdn.playwright.dev` を許可）か、ブラウザを
   事前インストール済みのイメージ／環境を使う。設定方法は
   https://code.claude.com/docs/en/claude-code-on-the-web を参照。
3. システムに既存の Chrome/Chromium がある場合は、`.mcp.json` の args に
   `"--browser", "chrome"`（または `--executable-path <path>`）を追加して既存ブラウザを使う。
