# 第 5 章：Skills スキルシステム

> **System Prompt、ツールホワイトリスト、パラメータ制約を再利用可能な設定としてパッケージする——この概念は異なるシステムで異なる実装があるが、核心の考え方は同じだ。**

---

## 用語説明

| 本章の用語 | 対応システム | 説明 |
|---------|---------|------|
| **Presets** | Shannon | ロールプリセット、`roles/presets.py` で定義 |
| **Agent Skills** | Anthropic オープン標準 | クロスプラットフォームのスキル規格、`.claude/skills/` など |

本章ではまず汎用概念を説明し、その後 Shannon Presets と Agent Skills の2つの実装をそれぞれ紹介する。

---

# Part A：汎用概念

## 5.1 スキルシステムとは？

前の章で、単体エージェント（Agent）のツールと推論能力について解説した。でもひとつ問題が見えてきた。同じエージェントでも、タスクが変わると途端に使えなくなる。

以前、あるクライアント向けにコードレビュー用のエージェントを作ったことがある。設定はシンプルだった。System Prompt で「潜在的なバグとセキュリティ問題を見つけろ」と指示して、ツールはファイル読み取りとコード検索だけ。結果は上々で、隠れた問題をかなり見つけてくれた。

1ヶ月後、クライアントから新しい要望が来た。「このエージェントで市場調査もできない？」

試してみたけど、まあ全然ダメだった。コードレビュー用の Prompt は「バグを探せ」「型安全性をチェックしろ」と言ってる。でも市場調査に必要なのは「トレンドを検索しろ」「データを比較しろ」「ソースを引用しろ」だ。ツールも合わない。ファイル読み取りは役に立たないし、必要なのはウェブ検索とデータ取得だ。

結局、午後いっぱいかけて「リサーチャー」ロールを一から作り直した。2つの設定、全く別物だ。

**これがスキルシステムで解決したい問題なんだ——あらかじめ用意したロール設定をワンクリックで切り替えられる。** コードレビューには `code_reviewer`、市場調査には `researcher` を使えばいい。

### 一言で定義

> **スキルシステム = System Prompt + ツールホワイトリスト + パラメータ制約のパッケージ**

![Skill 構造](assets/skill-structure.svg)

### なぜ必要？

1. **毎回の再設定を避ける**——タスクが変わっても Prompt を一から書き直す必要がない
2. **漏れやミスを減らす**——名前で参照するから、パラメータを忘れることがない
3. **チームでベストプラクティスを共有**——良い設定を再利用できる

### 2つの実装アプローチ

| アプローチ | 代表例 | 特徴 |
|------|------|------|
| **フレームワーク内蔵** | Shannon Presets | コードレベルの設定、Python 辞書 |
| **クロスプラットフォーム標準** | Agent Skills | ファイルレベルの設定、Markdown + YAML |

次にこの2つの実装をそれぞれ見ていこう。

---

# Part B：Shannon Presets

## 5.2 Shannon の Presets レジストリ

Shannon ではスキルシステムを Presets（プリセット）として実装していて、`roles/presets.py` に格納されている：

```python
_PRESETS: Dict[str, Dict[str, object]] = {
    "analysis": {
        "system_prompt": "You are an analytical assistant. Provide concise reasoning...",
        "allowed_tools": ["web_search", "file_read"],
        "caps": {"max_tokens": 30000, "temperature": 0.2},
    },
    "research": {
        "system_prompt": "You are a research assistant. Gather facts from authoritative sources...",
        "allowed_tools": ["web_search", "web_fetch", "web_crawl"],
        "caps": {"max_tokens": 16000, "temperature": 0.3},
    },
    "writer": {
        "system_prompt": "You are a technical writer. Produce clear, organized prose.",
        "allowed_tools": ["file_read"],
        "caps": {"max_tokens": 8192, "temperature": 0.6},
    },
    "generalist": {
        "system_prompt": "You are a helpful AI assistant.",
        "allowed_tools": [],
        "caps": {"max_tokens": 8192, "temperature": 0.7},
    },
}
```

3つのフィールドにはそれぞれ役割がある：

| フィールド | 役割 | 設計上の考慮点 |
|------|--------|----------|
| `system_prompt` | キャラ設定と行動規範を定義 | 具体的であるほど良い |
| `allowed_tools` | ツールのホワイトリスト | 最小権限の原則 |
| `caps` | パラメータ制約 | コストとスタイルを制御 |

### 安全なフォールバック

Preset を取得する関数には、いくつか注目すべきポイントがある：

```python
def get_role_preset(name: str) -> Dict[str, object]:
    key = (name or "").strip().lower() or "generalist"

    # エイリアスマッピング（後方互換性）
    alias_map = {
        "researcher": "research",
        "research_supervisor": "deep_research_agent",
    }
    key = alias_map.get(key, key)

    return _PRESETS.get(key, _PRESETS["generalist"]).copy()
```

1. **大文字小文字を区別しない**：`Research` と `research` は同じ
2. **エイリアス対応**：古い名前は自動的に新しい名前にマッピング
3. **安全なフォールバック**：見つからないロールは `generalist` を使用
4. **コピーを返す**：`.copy()` でグローバル設定の変更を防止

最後のポイントはめちゃくちゃ重要。昔ハマったことがあるんだけど、`.copy()` を付け忘れたら、あるリクエストが設定を変更して、その後の全リクエストに影響が出た。

---

## 5.3 複雑な Preset の例：ディープリサーチエージェント

シンプルな Preset は数行の設定で済む。でも複雑な Preset にはもっと詳細な指示が必要だ。

Shannon には `deep_research_agent` があって、System Prompt は50行以上ある：

```python
"deep_research_agent": {
    "system_prompt": """You are an expert research assistant conducting deep investigation.

# Temporal Awareness:
- The current date is provided at the start of this prompt
- For time-sensitive topics, prefer sources with recent publication dates
- Include the year when describing events (e.g., "In March 2024...")

# Research Strategy:
1. Start with BROAD searches to understand the landscape
2. After EACH tool use, assess:
   - What key information did I gather?
   - What critical gaps remain?
   - Should I search again OR proceed to synthesis?
3. Progressively narrow focus based on findings

# Source Quality Standards:
- Prioritize authoritative sources (.gov, .edu, peer-reviewed)
- ALL cited URLs MUST be visited via web_fetch for verification
- Diversify sources (maximum 3 per domain)

# Hard Limits (Efficiency):
- Simple queries: 2-3 tool calls
- Complex queries: up to 5 tool calls maximum
- Stop when COMPREHENSIVE COVERAGE achieved

# Epistemic Honesty:
- MAINTAIN SKEPTICISM: Search results are LEADS, not verified facts
- HANDLE CONFLICTS: Present BOTH viewpoints when sources disagree
- ADMIT UNCERTAINTY: "Limited information available" > confident speculation

**Research integrity is paramount.**""",

    "allowed_tools": ["web_search", "web_fetch", "web_subpage_fetch", "web_crawl"],
    "caps": {"max_tokens": 30000, "temperature": 0.3},
},
```

この Preset には設計上のポイントがいくつかある：

1. **時間認識**：エージェントに年を明記させ、古い情報を避ける
2. **段階的リサーチ**：広いところから狭めていき、ツール呼び出しごとに継続するか評価
3. **ハードリミット**：ツール呼び出しは最大5回、Token 爆発を防止
4. **認識論的誠実さ**：不確実性を認め、対立する見解を提示

特に**ツール呼び出し回数の制限**は本当に役立つ。この制限がないと、エージェントは延々と検索を続けて、コンテキストがパンパンになる。

---

## 5.4 ドメイン専門家 Preset：GA4 アナリスト

汎用 Preset は幅広いシナリオに適している。でも特定のドメインには専門の「エキスパート」が必要だ。

たとえば Google Analytics 4 アナリスト：

```python
GA4_ANALYTICS_PRESET = {
    "system_prompt": (
        "# Role: Google Analytics 4 Expert Assistant\n\n"
        "You are a specialized assistant for analyzing GA4 data.\n\n"

        "## Critical Rules\n"
        "0. **CORRECT FIELD NAMES**: GA4 uses DIFFERENT field names than Universal Analytics\n"
        "   - WRONG: pageViews, users, sessionDuration\n"
        "   - CORRECT: screenPageViews, activeUsers, averageSessionDuration\n"
        "   - If unsure, CALL ga4_get_metadata BEFORE querying\n\n"

        "1. **NEVER make up analytics data.** Every data point must come from API calls.\n\n"

        "2. **Check quota**: If quota below 20%, warn the user.\n"
    ),
    "allowed_tools": [
        "ga4_run_report",
        "ga4_run_realtime_report",
        "ga4_get_metadata",
    ],
    "provider_override": "openai",  # 特定のプロバイダーを指定可能
    "preferred_model": "gpt-4o",
    "caps": {"max_tokens": 16000, "temperature": 0.2},
}
```

ドメイン Preset には特殊な設定がある：

- `provider_override`：特定のプロバイダーを強制（たとえば特定タスクで GPT の方が効果的な場合）
- `preferred_model`：優先モデルを指定

これらは汎用 Preset にはない設定だ。

### 動的ツールファクトリ

ドメイン Preset にはもうひとつよくある要件がある：**設定に基づいてツールを動的に生成する**こと。

たとえば GA4 ツールは特定のアカウントにバインドする必要がある：

```python
def create_ga4_tool_functions(property_id: str, credentials_path: str):
    """アカウント設定に基づいて GA4 ツールを生成"""
    client = GA4Client(property_id, credentials_path)

    def ga4_run_report(**kwargs):
        return client.run_report(**kwargs)

    def ga4_get_metadata():
        return client.get_available_dimensions_and_metrics()

    return {
        "ga4_run_report": ga4_run_report,
        "ga4_get_metadata": ga4_get_metadata,
    }
```

こうすれば、異なるユーザーが異なる GA4 アカウントを使える。同じ Preset だけど、異なる認証情報をバインドできるわけだ。

---

## 5.5 Prompt テンプレートのレンダリング

同じ Preset でもシナリオに応じて異なる変数を注入したいことがある。

たとえばデータ分析 Preset：

```python
"data_analytics": {
    "system_prompt": (
        "# Setup\n"
        "profile_id: ${profile_id}\n"
        "User's account ID: ${aid}\n"
        "Date of today: ${current_date}\n\n"
        "You are a data analytics assistant..."
    ),
    "allowed_tools": ["processSchemaQuery"],
}
```

呼び出し時にパラメータを渡す：

```python
context = {
    "role": "data_analytics",
    "prompt_params": {
        "profile_id": "49598h6e",
        "aid": "7b71d2aa-dc0d-4179-96c0-27330587fb50",
        "current_date": "2026-01-03",
    }
}
```

レンダリング関数が `${variable}` を実際の値に置き換える：

```python
def render_system_prompt(prompt: str, context: Dict) -> str:
    variables = context.get("prompt_params", {})

    def substitute(match):
        var_name = match.group(1)
        return str(variables.get(var_name, ""))

    return re.sub(r"\$\{(\w+)\}", substitute, prompt)
```

レンダリング後：

```
# Setup
profile_id: 49598h6e
User's account ID: 7b71d2aa-dc0d-4179-96c0-27330587fb50
Date of today: 2026-01-03

You are a data analytics assistant...
```

---

## 5.6 実行時の動的拡張

Preset が定義するのは静的な設定だ。でも実行時には動的にコンテンツが注入される：

```python
# 現在日付を注入
current_date = datetime.now().strftime("%Y-%m-%d")
system_prompt = f"Current date: {current_date} (UTC).\n\n" + system_prompt

# 言語指示を注入
if context.get("target_language") and context["target_language"] != "English":
    lang = context["target_language"]
    system_prompt = f"CRITICAL: Respond in {lang}.\n\n" + system_prompt

# リサーチモード拡張
if context.get("research_mode"):
    system_prompt += "\n\nRESEARCH MODE: Do not rely on snippets. Use web_fetch to read full content."
```

こうして Preset の静的設定と実行時コンテキストが組み合わさって、最終的に LLM に送られる System Prompt になる。

---

## 5.7 Vendor Adapter パターン

外部システムと深く統合する必要がある Preset には、Shannon は巧みな設計を使っている：

```
roles/
├── presets.py              # 汎用プリセット
├── ga4/
│   └── analytics_agent.py  # GA4 専用
├── ptengine/
│   └── data_analytics.py   # Ptengine 専用
└── vendor/
    └── custom_client.py    # クライアントカスタム（コミットしない）
```

読み込みロジック：

```python
# オプションで vendor ロールを読み込み
try:
    from .ga4.analytics_agent import GA4_ANALYTICS_PRESET
    _PRESETS["ga4_analytics"] = GA4_ANALYTICS_PRESET
except Exception:
    pass  # モジュールが存在しない場合はサイレントに失敗

try:
    from .ptengine.data_analytics import DATA_ANALYTICS_PRESET
    _PRESETS["data_analytics"] = DATA_ANALYTICS_PRESET
except Exception:
    pass
```

メリットは：

1. **コアコードがきれい**：汎用 presets は vendor モジュールに依存しない
2. **優雅なフォールバック**：モジュールが存在しなくてもエラーにならない
3. **クライアントカスタマイズ**：プライベートな vendor ディレクトリにコミットしないコードを格納できる

---

## 5.8 新しい Preset の設計

「コードレビュアー」Preset を作るとしたら、どう設計する？

```python
"code_reviewer": {
    "system_prompt": """You are a senior code reviewer with 10+ years of experience.

## Mission
Review code for bugs, security issues, and maintainability problems.
Focus on HIGH-IMPACT issues that matter for production.

## Severity Levels
1. CRITICAL: Security vulnerabilities, data corruption risks
2. HIGH: Logic errors, race conditions, resource leaks
3. MEDIUM: Code smells, performance issues
4. LOW: Style, naming, documentation

## Output Format
For each issue:
- **Severity**: CRITICAL/HIGH/MEDIUM/LOW
- **Location**: file:line
- **Issue**: Brief description
- **Suggestion**: How to fix
- **Confidence**: HIGH/MEDIUM/LOW

## Rules
- Only report issues with MEDIUM+ confidence
- Limit to 10 most important issues per review
- Skip style issues unless explicitly asked

## Anti-patterns to Watch
- SQL injection, XSS, command injection
- Hardcoded secrets in code
- Unchecked null access
- Resource leaks
""",
    "allowed_tools": ["file_read", "grep_search"],
    "caps": {"max_tokens": 8000, "temperature": 0.1},
}
```

設計の判断：

| 判断 | 理由 |
|------|------|
| 低い temperature (0.1) | コードレビューは正確さが重要、創造性は不要 |
| 10件に制限 | 情報過多を避ける |
| 確信度の表示 | どれを優先的に確認すべきかユーザーに伝える |
| 最小限のツールセット | ファイル読み取りと検索だけで十分、書き込みは不要 |

---

## 5.9 よくある落とし穴

### 落とし穴 1：System Prompt が曖昧すぎる

```python
# 曖昧すぎる - 具体性がない
"system_prompt": "You are a helpful assistant."

# 具体的で明確
"system_prompt": """You are a research assistant.

RULES:
- Cite sources for all factual claims
- Use bullet points for readability
- Maximum 3 paragraphs unless asked for more

OUTPUT FORMAT:
## Summary
[1-2 sentences]

## Key Findings
- Finding 1 (Source: ...)
"""
```

### 落とし穴 2：ツール権限が広すぎる

```python
# 権限が広すぎる - ツールを与えすぎ
"allowed_tools": ["web_search", "file_write", "shell_execute", "database_query"]

# 最小権限 - 必要なものだけ
"allowed_tools": ["web_search", "web_fetch"]  # リサーチタスクには検索だけで十分
```

ツールを与えすぎると、LLM が混乱する（どれを使えばいいかわからない）し、セキュリティリスクも増える。

### 落とし穴 3：パラメータ制約を設定しない

```python
# 制限なし - 制御不能になりやすい
"caps": {}

# タスクに応じて制約を設定
"caps": {"max_tokens": 1000, "temperature": 0.3}  # 短い応答
"caps": {"max_tokens": 16000, "temperature": 0.6}  # 長文生成
```

`max_tokens` を設定しないと、Token 消費が制御不能になる。

### 落とし穴 4：フォールバック戦略がない

```python
# モジュールが存在しないとクラッシュ
from .custom_module import CUSTOM_PRESET
_PRESETS["custom"] = CUSTOM_PRESET

# 優雅なフォールバック
try:
    from .custom_module import CUSTOM_PRESET
    _PRESETS["custom"] = CUSTOM_PRESET
except Exception:
    pass  # デフォルトの generalist を使用
```

---

# Part C：Agent Skills

## 5.10 Agent Skills：コンテキスト膨張問題の解決

前半で Shannon の Presets を見た。ここからはもうひとつのスキルシステム、Agent Skills を見ていこう。

### 問題：コンテキストウィンドウは希少資源

2025年、AI プログラミングツールが爆発的に増えた。Claude Code、Cursor、GitHub Copilot、Codex CLI……開発者はすぐにある問題に気づいた：**コンテキストウィンドウが足りない**。

MCP（Model Context Protocol）を例に挙げよう。MCP はエージェントを外部サービス——GitHub、Jira、データベース——に接続できる。素晴らしく聞こえるが、代償がある：

| MCP サーバー | ツール数 | Token 消費 |
|-----------|---------|-----------|
| GitHub 公式 | 93 ツール | ~55,000 tokens |
| Task Master | 59 ツール | ~45,000 tokens |

ある Claude Code ユーザーが報告した：いくつかの MCP を有効にしたら、コンテキスト使用量が 178k/200k（89%）に達した。そのうち MCP ツール定義だけで 63.7k を占めていた。まだ何も始めてないのに、コンテキストがほぼ一杯だ。

問題の根本は：**MCP は起動時にすべてのツール定義を読み込む**ということ。使うかどうかに関係なく、93 個の GitHub ツールのスキーマがすべてコンテキストに詰め込まれる。

### Skills の解決策：段階的開示

2025年10月、Anthropic は Claude Code に Skills を導入した。核心の設計思想は：**全量読み込みではなく、必要に応じて読み込む**。

公式はこれを「段階的開示」（Progressive Disclosure）と呼んでいて、整理された本に例えている：

> 「まず目次、次に具体的な章、最後に詳細な付録。」

技術的には3層に分かれている：

1. **メタデータ層**：起動時は `name` と `description` だけを読み込む。各 Skill 約 30-50 tokens
2. **コンテンツ層**：ユーザーリクエストがマッチしたときだけ、完全な `SKILL.md` を読み込む。通常 < 5k tokens
3. **拡張層**：参照される `reference.md`、`examples/`、`scripts/` は実際に必要なときだけ読み込む

効果は？**何百もの Skills をインストールできるが、起動時には数千 tokens しか消費しない**。公式ドキュメントの表現では："the amount of context that can be bundled into a skill is effectively unbounded"（スキルにバンドルできるコンテキスト量は実質的に無制限）。

### MCP との関係

Skills は MCP を置き換えるものではなく、補完するものだ：

- **MCP は「パイプ」**——外部サービスに接続する API
- **Skills は「マニュアル」**——エージェントにこれらの API を使ってタスクを完了する方法を教える

例を挙げよう：MCP で Jira に接続したが、エージェントは「スプリントを作成」するにはどのエンドポイントを呼び出すか、どんなパラメータを渡すか知らない。このとき「Jira プロジェクト管理」Skill が必要で、完全なワークフローを教える。

そして Skills 自体の Token 効率も、MCP がもたらすコンテキスト圧迫を緩和する——MCP 接続は大量の tokens を占有するが、Skill の指示は必要なときだけ読み込まれる。

### タイムライン

| 時期 | イベント |
|------|------|
| 2025年2月 | Claude Code リリース |
| 2025年10月 | Claude Code に Skills 機能導入；Simon Willison の記事が注目を集める |
| 2025年12月 | OpenAI Codex CLI が Skills サポートを追加；Anthropic がオープン標準を発表 |
| 2026年1月 | Google Antigravity、Cursor などが追随 |

---

## 5.11 Agent Skills フォーマット仕様

### ディレクトリ構造

Skill はディレクトリで、`SKILL.md` がエントリーポイント：

```
my-skill/
├── SKILL.md           # メイン指示（必須）
├── template.md        # テンプレートファイル（オプション）
├── reference.md       # 詳細参考ドキュメント（オプション）
├── examples/
│   └── sample.md      # サンプル出力（オプション）
└── scripts/
    └── helper.py      # 実行可能スクリプト（オプション）
```

`SKILL.md` は必須で、その他のファイルは必要に応じて追加する。

### SKILL.md フォーマット

```yaml
---
name: my-skill
description: このスキルが何をするか、いつ使うか
allowed-tools: Read, Grep, Glob
---

## あなたの指示

このタスクを実行するとき：
1. 第一ステップ...
2. 第二ステップ...
```

### Frontmatter フィールド

| フィールド | 必須 | 説明 |
|------|------|------|
| `name` | いいえ | スキル名、デフォルトはディレクトリ名。小文字、数字、ハイフン |
| `description` | 推奨 | Claude がいつ自動読み込みするか判断するために使用 |
| `allowed-tools` | いいえ | ツールホワイトリスト、スキルが使えるツールを制限 |
| `disable-model-invocation` | いいえ | `true` に設定すると Claude の自動呼び出しを禁止 |
| `user-invocable` | いいえ | `false` に設定すると `/` メニューから非表示 |
| `context` | いいえ | `fork` に設定するとサブエージェントで実行 |
| `agent` | いいえ | サブエージェントタイプを指定（`Explore`、`Plan` など） |

### 呼び出し制御

2つのフィールドが誰がスキルを呼び出せるかを制御する：

- `disable-model-invocation: true`：ユーザーだけが呼び出せる（副作用のある操作、たとえばデプロイに適している）
- `user-invocable: false`：Claude だけが呼び出せる（バックグラウンド知識に適している、ユーザーが直接トリガーする必要がない）

### 高度な機能

**変数置換**：

```yaml
---
name: fix-issue
description: GitHub issue を修正
---

GitHub issue $ARGUMENTS を修正：
1. issue の説明を読む
2. 修正を実装
3. commit を作成
```

`/fix-issue 123` を実行すると、`$ARGUMENTS` は `123` に置き換えられる。

**動的コンテキスト注入**：

```yaml
---
name: pr-summary
description: PR の変更を要約
---

## PR コンテキスト
- PR diff: !`gh pr diff`
- PR コメント: !`gh pr view --comments`

## タスク
この PR の変更を要約...
```

`` !`command` `` 構文はまずコマンドを実行し、出力を Skill コンテンツに注入する。

**スクリプト実行**：

Skills は Python や Bash スクリプトを含むことができ、Claude がそれらを実行できる：

```
my-skill/
├── SKILL.md
└── scripts/
    └── analyze.py    # Claude がこのスクリプトを実行できる
```

---

## 5.12 シンプルな例

「コードレビュー」スキルを作成しよう。Skills ディレクトリに `code-review/SKILL.md` を新規作成：

```yaml
---
name: code-review
description: コードをレビューし、バグ、セキュリティ問題、保守性の問題を見つける。ユーザーが「review」「レビュー」「このコードを見て」と言ったときに使用。
allowed-tools: Read, Grep, Glob
---

## レビュー基準

1. **セキュリティ問題**（最優先）
   - SQL インジェクション、XSS、コマンドインジェクション
   - ハードコードされた秘密鍵

2. **ロジックエラー**
   - null ポインタ、範囲外アクセス、リソースリーク

3. **保守性**
   - コード重複、長すぎる関数、不明確な命名

## 出力フォーマット

各問題について：
- **重要度**：CRITICAL / HIGH / MEDIUM / LOW
- **場所**：file:line
- **問題**：簡潔な説明
- **提案**：修正方法

## ルール

- MEDIUM 以上の確信度の問題のみ報告
- 最大 10 件の最重要問題を報告
- 明示的に求められない限り、純粋なスタイル問題はスキップ
```

**テスト方法**：

- 自動トリガー：「このコードをレビューして」と言うと、エージェントが自動的にマッチして読み込む
- 手動トリガー：`/code-review src/auth/` と入力

---

## 5.13 公式リソースとエコシステム

### 公式リソース

| リソース | リンク | 説明 |
|------|------|------|
| Agent Skills 仕様 | [agentskills.io](https://agentskills.io) | 公式標準定義と SDK |
| Anthropic Skills リポジトリ | [github.com/anthropics/skills](https://github.com/anthropics/skills) | 公式サンプル集 |
| Claude Code ドキュメント | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) | 使用ガイド |
| Skills Directory | [claude.com/connectors](https://claude.com/connectors) | パートナー Skills ディレクトリ |

### Skills Directory

2025年12月、Anthropic は Skills Directory も同時にリリースした——ユーザーがパートナーが構築した Skills を閲覧・有効化できるスキル配布プラットフォームだ。

最初のパートナーには以下が含まれる：

| パートナー | 提供する Skills |
|---------|--------------|
| **Atlassian** | Jira と Confluence 統合——要件ドキュメントをタスクに変換、ステータスレポート生成、社内ナレッジベース検索 |
| **Figma** | デザイン理解——Claude が Figma デザインのコンテキスト、詳細、意図を理解し、正確にコードに変換 |
| **Notion** | ドキュメントとデータベース操作 |
| **Canva** | デザインリソース生成 |
| **Stripe** | 決済統合ワークフロー |
| **Zapier** | 自動化接続 |
| **Vercel** | デプロイワークフロー |
| **Cloudflare** | エッジコンピューティング設定 |

これらの Skills は対応する MCP コネクターと併用できる——MCP が API 接続を提供し、Skill がワークフロー知識を提供する。

---

## 5.14 Agent Skills の Multi-Agent オーケストレーション設計

Shannon も Agent Skills をサポートしている。前述の Agent Skills 標準は主にシングル Agent シナリオ向けだ——1つの Agent が1つの Skill を読み込み、ステップに沿って実行する。でも Multi-Agent システムでは問題が変わる：**Orchestrator はタスクをシングル Agent に Skill に従って実行させるべきか、それとも複数の Agent に分割して協力させるべきか、どう判断するのか？**

Shannon の設計上の答えはシンプルだ：**Skill 自身に宣言させる。**

### Skill がオーケストレーション経路を決定する

Shannon は Anthropic 標準の上にキーフィールドを追加した：`requires_role`。このフィールドは単にロールを指定するだけでなく、Orchestrator のルーティング判断に直接影響する：

- **Skill が `requires_role` を宣言している** → Orchestrator は LLM タスク分解をスキップし、シングル Agent 実行プランを作成する。Skill 自体が完全なワークフローステップを定義しているため、分割するとかえって矛盾が生じる。

- **Skill が role を宣言していない** → Orchestrator は通常通り LLM でタスク分解を行い、サブタスクに分割して DAG 並列実行する。

つまり、**`requires_role` は Skills と Multi-Agent オーケストレーションの分岐点**だ。Skill の作者が設計時にタスクの実行モードを決定する。

なぜこの設計なのか？タスクごとに協力パターンが根本的に異なるからだ。

コードレビュー、デバッグ、TDD——これらのタスクは本質的に一人の専門家が最初から最後まで担当する必要がある。複数の Agent に分割するとコンテキストが失われる。一方、「X 分野の最新動向を調査する」のようなタスクは、複数の Agent が並列で検索・集約するのが自然だ。

**Skill の作者がタスクの特性を最もよく理解している。だから Skill 自身に実行モードを決定させる。**

これは Presets と Skills の関係も明らかにする——**Presets は能力を管理し、Skills はワークフローを管理する**。Skill は `requires_role` を通じて Preset を参照する。たとえば `code-review` Skill は `requires_role: critic` を指定し、実行時に Agent は読み取り専用アクセスのみ（`critic` Preset は `file_read` のみ許可）。Skill の Markdown 本文は具体的な3段階ワークフローを定義する：コンテキスト収集 → 分析（セキュリティ/品質/パフォーマンス）→ レポート出力。

この分離のメリットは**自由に組み合わせられること**：同じ `critic` Preset に `code-review`、`architecture-review`、`dependency-audit` など異なる Skill を組み合わせられる。能力の境界は変わらず、ワークフローがタスクに応じて切り替わる。

### セキュリティ設計：3層の重ね合わせ

Multi-Agent システムでは、セキュリティ境界はシングル Agent よりも重要だ——暴走した1つの Agent がオーケストレーションチェーン全体に影響しかねない。Shannon は Skill レベルで3層の防御を重ねている：

1. **誰が使えるか**：`dangerous: true` の Skill は admin/owner 権限または専用の `skills:dangerous` 認可スコープが必要
2. **どのツールが使えるか**：`requires_role` が参照する Preset がツールホワイトリストを制限
3. **Token をどれだけ使うか**：`budget_max` が1回の実行あたりの Token 消費上限を制限

3層が独立して制御され、相互依存しない。ある Skill は non-dangerous でもツール制限が厳しい（`critic`）こともあれば、dangerous でもツール権限が広い（例：本番環境デプロイ）こともある。

### Agent 間ネゴシエーションにおける Skills の役割

Multi-Agent 協調では、Agent 間でタスクを受け渡す必要がある。Shannon の P2P メッセージプロトコルには `Skills` フィールドがある——送信側が「このタスクを完了するには `code-review` スキルが必要」と宣言でき、受信側はこれに基づいて自分が引き受けられるか判断する。

これは Skills が単一の Agent のやり方をガイドするだけでなく、システムが**誰がやるか**を決定する助けにもなることを意味する。Part 5（Multi-Agent オーケストレーション）の Handoff メカニズムでこのトピックをさらに展開する。


---

## 本章を超えて：Tools、MCP、Skills の統一的視点

Skills の説明を終えて、一歩引いて Part 2 全体の概念がどう関連しているか見てみよう。

**本質**：Tools、MCP、Skills はすべてエージェントのコンテキストに情報を注入し、エージェントの能力を補完するものだ。

| メカニズム | 何を注入するか | 何の能力を補完するか |
|------|---------|-------------|
| Tools | 関数定義 + 実行ロジック | 外部システムとのインタラクション |
| MCP | ツール定義（外部サービスから） | 外部サービスへの接続 |
| Skills | 指示 + ワークフロー知識 | ドメイン専門知識 |

三者の関係：

```
Tools ← 基本能力ユニット
  ↑
MCP ← 外部サービスが Tools を公開する標準的な方法
  ↑
Skills ← エージェントに Tools を組み合わせてタスクを完了する方法を教える
```

**設計上の共通制約**：コンテキストウィンドウは希少資源。

だからどう変化しても、設計では以下が必要：

- **必要に応じて読み込む**——使わないものは入れない
- **Token 消費を最小化**——メタデータ先行、コンテンツ遅延
- **組み合わせ可能**——小さなモジュールを組み合わせて大きな能力に
- **最小権限**——タスク完了に必要なツールだけを与える

この4つの原則が Part 2 のすべての章を貫いている。

### 本質を理解してこそ、エコシステムを使いこなせる

Skills のエコシステムは確かに急速に発展している。クロスプラットフォーム標準、数十のサポートツール、パートナーディレクトリ、さらには skill-creator skill が skill を書いてくれる——参入障壁はどんどん下がっている。

でも、エコシステムが繁栄しているからといって、そのまま使えるわけじゃない。

本質に立ち返ろう：**Skills は構造化されたコンテキスト注入に過ぎない**。「エージェントに仕事を教える」コストは下がったけど、何を教えるか、どう教えるかは自分で考えないといけない。

市場の汎用 Skills は出発点として参考になるけど、本当に価値を生むのは：
- 自社の内部プロセスとベストプラクティス
- クライアント固有のシナリオとニーズ
- チームが蓄積した領域のノウハウ

Skills Directory の Atlassian、Figma、Stripe が価値あるのは、SKILL.md のフォーマットのおかげじゃない。彼らが長年の製品経験と領域知識をそこにエンコードしたからだ。

**アドバイス**：エコシステムの Skills でフォーマットと考え方を学び、コアとなる差別化された Skills は自分で蓄積しよう。

---

## 本章のまとめ

1. **スキルシステム = System Prompt + ツールホワイトリスト + パラメータ制約**——ロール設定を再利用可能な単位にパッケージ

2. **Shannon は Presets で実装**——Python 辞書、`roles/presets.py` に格納、フレームワークと深く統合

3. **Agent Skills は段階的開示でコンテキスト膨張を解決**——起動時はメタデータのみ読み込む（30-50 tokens/skill）、コンテンツは必要に応じて読み込む

4. **Agent Skills のフォーマットはシンプル**——ディレクトリ + SKILL.md + オプションのサポートファイル

5. **Skills と MCP は補完関係**——MCP が API 接続を提供、Skills がワークフロー指示を提供

---

## Shannon Lab（10分で体験）

このセクションでは、10分で本章の概念を Shannon ソースコードに対応づける。

### 必読（1ファイル）

- [`roles/presets.py`](https://github.com/Kocoro-lab/Shannon/blob/main/python/llm-service/llm_service/roles/presets.py)：`_PRESETS` 辞書を見て、ロールプリセットの構造を理解する。`deep_research_agent` という複雑な例に注目

### 選読（興味に応じて選択）

- [`config/skills/core/code-review.md`](https://github.com/Kocoro-lab/Shannon/blob/main/config/skills/core/code-review.md)：完全な内蔵 Skill を見る、`requires_role: critic` と `budget_max: 5000` の組み合わせに注目
- [`go/orchestrator/internal/skills/`](https://github.com/Kocoro-lab/Shannon/tree/main/go/orchestrator/internal/skills)：Skills レジストリの Go 実装、`models.go`（Skill 構造体）と `registry.go`（読み込みロジック）に注目
- [`roles/ga4/analytics_agent.py`](https://github.com/Kocoro-lab/Shannon/blob/main/python/llm-service/llm_service/roles/ga4/analytics_agent.py)：実際のベンダーカスタムロールを見る
- `research` と `analysis` の2つのプリセットを比較して、なぜツールリストが異なるか考える

---

## 演習

### 演習 1：既存の Preset を分析する

Shannon の `presets.py` を読んで、以下に答えよ：

1. `research` と `analysis` の2つのロールはどう違う？
2. なぜ `writer` ロールの temperature は `analysis` より高い？
3. `generalist` ロールの `allowed_tools` はなぜ空のリスト？

### 演習 2：Preset を設計する

「コードレビュー」タスク用の Preset を設計せよ：

1. System Prompt を書く（少なくとも：職責、レビュー基準、出力形式を含む）
2. 必要なツールをリストアップ（file_read？git_diff？その他？）
3. temperature と max_tokens を設定（理由も説明）

### 演習 3：Agent Skill を作成する

`~/.claude/skills/` にカスタム Skill を作成：

1. よく行うタスクを選ぶ（ドキュメント作成、テスト生成、コードリファクタリング...）
2. `SKILL.md` を書く、frontmatter と指示を含める
3. Claude Code でテスト

### 演習 4（発展）：2つの実装を比較する

考えてみよう：Shannon Presets と Agent Skills はそれぞれどんなシナリオに適している？

- どんなときにコードレベルの設定（Presets）の方が良い？
- どんなときにファイルレベルの設定（Skills）の方が良い？

---

## 参考資料

- [Agent Skills 公式仕様](https://agentskills.io) - クロスプラットフォーム標準定義
- [Anthropic エンジニアリングブログ：Equipping agents for the real world](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) - Agent Skills 設計思想
- [Simon Willison: Claude Skills are awesome](https://simonwillison.net/2025/Oct/16/claude-skills/) - Skills がなぜ重要か
- [Claude Code Skills ドキュメント](https://code.claude.com/docs/en/skills) - 使用ガイド
- [Shannon Roles Source Code](https://github.com/Kocoro-lab/Shannon/tree/main/python/llm-service/llm_service/roles) - Presets コード実装

---

## 次章の予告

スキルシステムは「エージェントがどう振る舞うべきか」という問題を解決した。でもまだ別の問題がある。

エージェントがタスクを実行しているとき、何をしているか知りたいよね。重要なポイントでカスタムロジックを挿入したいこともある。

たとえば：
- ツール呼び出しごとにログを記録
- Token 消費が閾値を超えたら警告を出す
- 特定の操作の前にユーザーの確認を求める

これが次章の内容——**Hooks とイベントシステム**だ。

次の章で続けよう。
