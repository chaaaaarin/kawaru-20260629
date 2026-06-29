# No.42 Codex トークン節約術

## 動画概要
OpenAI Codex を使っていてトークン（＝料金・レート制限）が増えがちな理由と、今日から使える節約テクニックを体系的に解説する回。「コンテキストを小さく保つ＝節約＆精度向上の一石二鳥」が核心メッセージ。Claude Code版（No.71）と対比しながら、Codex固有の仕組みと節約術を整理する。

## 3行まとめ
1. Codex は毎メッセージで会話全履歴を読み直す → 長い会話ほど1回のコストが雪だるま式に増える
2. /clear（スレッド切替）・/compact（コンテキスト圧縮）・AGENTS.md最適化の3点セットで劇的に改善できる
3. モデル選択（mini ファースト）・MCPサーバー整理・.codexignore の3つが最大の節約ポイント

---

## セクション構成案

### ① なぜトークンが増えるのか（仕組み）
- Codex はリクエストのたびに会話全履歴をまるごと送る（LLM共通の仕組み）
- メッセージ1は1x、メッセージ10は10x、メッセージ30は31x（線形以上に増加）✅
- 「auth.py のバグを直して」という1タスクだけで18,500トークン消費 🔶
  - auth.py 500行読み込み: 5,000トークン
  - 関連インポート3ファイル: 9,000トークン
  - コードベース構造分析: 2,000トークン
  - 修正生成+書き込み: 1,500トークン
  - コンテキストオーバーヘッド: 1,000トークン

### ② 何がコンテキストを消費するか
| 要素 | コスト | 備考 |
|------|--------|------|
| システムプロンプト + AGENTS.md | 2,000〜5,000トークン/ターン | プロンプトキャッシュが効く |
| MCPサーバー（ツール定義） | 550〜1,400トークン/ツール | GitHub MCP（93ツール）= 55,000トークン/ターン |
| ファイル読み込み | 500行 ≈ 15,000トークン | 自動トランケートなし |
| 推論トレース（xhigh） | mediumの3〜5倍 | 画面非表示でも課金される |
| 会話履歴 | ターンを重ねるほど増大 | セッション全体に累積 |

### ③ 節約の3軸
- **軸①** コマンド操作（/clear・/compact・/status・reasoning effort）
- **軸②** 設定ファイル最適化（AGENTS.md・Skills・.codexignore・MCPサーバー）
- **軸③** プロセス最適化（モデル選択・プロンプトの絞り込み・エージェント設計）

### ④ 軸①：コマンド操作
- `/clear` ← **最重要・最簡単**。タスクが変わったら新スレッドで開始 ✅
- `/compact [指示]` ← **60%でやるのが最善**。自動発火は95%なので手遅れになりやすい 🔶
  - カスタム指示で何を残すか指定できる（例: 「認証リファクタと失敗テストを重点的に」）
- `/status` ← コンテキスト使用量をパーセントで確認 ✅
- `reasoning effort` ← デフォルトはauto。ルーティン作業は `low`、難問のみ `high/xhigh` ✅
  - xhigh は same prompt で medium の3〜5倍消費。変数名変更に xhigh は不要

**パターン比較（スレッド切替のタイミング）**
```
パターンA: バグ修正→次の機能実装（同一スレッド継続）→履歴が重複コンテキストを増やし続ける
パターンB: バグ修正終了→/clear→次の機能実装（最良）
```

### ⑤ 軸②：設定ファイル最適化
- **AGENTS.md は簡潔に（2KiB以下が理想）** ✅ 毎ターン全読み込みされる。プロンプトキャッシュで2回目以降10%で済むが、長ければ長いほど損
- 繰り返しワークフローは **Skills へ移行** → idle時は50〜100トークンのみ、使うときだけフルロード ✅
- **.codexignore を作る** → node_modules / dist / *.lock / __pycache__ 等を除外 🔶
- **MCPサーバーは使わないものを無効化** ← GitHub MCP 1本で55,000トークン/ターン！🔶

```
# .codexignore 推奨設定
node_modules/
dist/
*.lock
__pycache__/
.git/
backend/static/
```

### ⑥ 軸③：プロセス最適化
- **モデル選択（最大の節約ポイント）**
  - mini: 検索・フォーマット・テスト生成・ドキュメント ← gpt-5.4の1/5コスト 🔶
  - gpt-5.4: 複雑なマルチファイル作業・バグ診断
  - reasoning models (o4-mini/o3): セキュリティ監査・難しいアルゴリズムのみ ← o3はgpt-5.4の15倍
- **プロンプトを絞る**（vague → targeted で60%削減）🔶
  - NG: 「認証モジュールをよりセキュアにリファクタ」（全体スキャンが走る）
  - OK: 「/app/auth/login.py の authenticate() にレート制限を追加。Redis クライアントを使い5回/15分/IP」
- **1プロジェクト×1スレッド原則** ← 無関係なリポジトリを開きっぱなしにしない 🔶
- **並列エージェント vs 逐次**
  - 独立タスク: 並列OK（各エージェントが独立したworktreeで動く）✅
  - 共有コンテキストのタスク: 逐次が節約（並列は文脈共有ができずやり直しが増える）🔶
  - 3〜4並列→逐次で40%削減の事例あり 🔶

### ⑦ 数字で見る節約効果
- .codexignore 適用: 40〜60%削減 🔶（複数記事）
- /compact を60%で手動実行: 30〜50%削減（subsequent calls）🔶
- targeted vs vague prompt: 60%削減 🔶（BSWEN実測）
- 並列→逐次実行: 40%削減 🔶（BSWEN: 50K→30K）
- mini vs full model: 1タスクあたり85%節約 🔶（価格表から計算）
- MCP無効化: 最大55,000トークン/ターン節約 🔶（GitHub MCP例）
- Codex CLI vs Claude Code: Codexは約4分の1トークン消費 🔶（MindStudio実測）

### ⑧ Claude Code との比較
| 観点 | Codex CLI | Claude Code |
|------|-----------|------------|
| 典型的なトークン消費 | 少ない（黙って実装） | 多い（詳細説明・確認） |
| スタイル | 指示に従い黙々と実行 | 設計・判断・説明も行う |
| 向いているタスク | 実装・リファクタ・繰り返し処理 | 複雑な設計・デバッグ・探索 |
| /compact | 60%で手動推奨 | 同様 |
| 設定ファイル | AGENTS.md | CLAUDE.md |
| 除外ファイル | .codexignore | .claudeignore |

### ⑨ 今日からやること
1. 次のタスクで /clear を使ってスレッドを切り替えてみる
2. .codexignore を作成して node_modules 等を除外する
3. /status でコンテキスト使用量を確認し、60%で /compact を試す

---

## 確度表記

### ✅ 公式確認済み
- AGENTS.md は毎ターン全読み込みされる（公式ドキュメント）
- Skills は idle 時は名前・説明のみ、使用時フルロード（公式）
- /compact・/clear・/status コマンドが存在（公式CLIドキュメント）
- 推論トレースは画面非表示でも出力トークンとして課金（公式）
- 5時間ローリングウィンドウでリセット（月次ではない）（公式ヘルプ）
- プロンプトキャッシュ: cached input は uncached の約10%（公式）

### 🔶 一次情報あり（記事・ベンチマーク）
- GitHub MCP（93ツール）= 55,000トークン/ターン（danielvaughan.com記事）
- auth.py fix = 18,500トークン（BSWEN実測）
- targeted vs vague prompt で60%削減（BSWEN実測）
- 並列→逐次で40%削減（BSWEN実測）
- mini vs full model ≈ 1/5コスト（openai.com/pricing から計算）
- .codexignore で40〜60%削減（複数記事）
- Codex CLI vs Claude Code ≈ 4x トークン差（MindStudio実測, TypeScriptタスク: 234K vs 72K）
- /compact at 60% で30〜50%削減（danielvaughan.com記事）

### ⚠️ 未確認・注意が必要
- GPT-5.3-Codex-Spark は別クォータで「実質無料」→ Research Preview 期間限定。現在も有効か要確認
- 上記の削減パーセンテージはすべて記事ベースの実測値。条件が違えば変わる可能性あり
- モデル名（GPT-5.4, GPT-5.3-Codex 等）は記事執筆時点の名称で、現在のUI表示と異なる場合あり

---

## 出典

**公式**
- https://developers.openai.com/codex/learn/best-practices
- https://developers.openai.com/codex/guides/agents-md
- https://developers.openai.com/codex/subagents
- https://developers.openai.com/codex/cli/slash-commands
- https://developers.openai.com/codex/cli/features
- https://developers.openai.com/codex/pricing
- https://help.openai.com/en/articles/20001106-codex-rate-card
- https://help.openai.com/ja-jp/articles/20001106-codex-rate-card
- https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide
- https://openai.com/index/introducing-codex/

**日本語記事**
- https://ai-rise.net/column/codex-token-saving/
- https://ai-rise.net/column/codex-token-recovery-guide/
- https://qiita.com/Koukyosyumei/items/4e1a426ba09a49ca2421
- https://note.com/ukisystem/n/nacd123d296da
- https://note.com/id_helpdesk/n/n13d0f148600c
- https://uravation.com/media/openai-codex-pricing-complete-guide-2026/
- https://uravation.com/media/codex-plan-mode-complete-guide-2026/

**英語記事**
- https://dev.to/stevengonsalvez/token-optimisation-101-stop-burning-money-on-ai-coding-agents-4mce
- https://docs.bswen.com/blog/2026-03-26-codex-usage-optimization/
- https://iceberglakehouse.com/posts/2026-03-context-openai-codex/
- https://codex.danielvaughan.com/2026/03/31/codex-cli-context-compaction-architecture/
- https://codex.danielvaughan.com/2026/03/28/codex-cli-cost-management-token-strategy/
- https://codex.danielvaughan.com/2026/04/08/codex-cli-performance-optimization/
- https://codex.danielvaughan.com/2026/04/20/codex-cli-context-window-budget-token-management-large-codebases/
- https://www.mindstudio.ai/blog/codex-vs-claude-code-context-window-token-efficiency
- https://medium.com/@anshulbansal930/how-to-optimize-token-usage-in-codex-44075340eae9
- https://portkey.ai/blog/openai-codex-best-practices/

**GitHub**
- https://github.com/openai/codex/blob/main/AGENTS.md
- https://github.com/alexgreensh/token-optimizer
- https://github.com/Austin1serb/agents-md
