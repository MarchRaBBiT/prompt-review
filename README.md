# prompt-review

Claude Code / Codex のスキルとして動作する、AIエージェント対話履歴の分析ツール。
過去のプロンプトから技術理解度・プロンプティングパターン・AI依存度を推定し、日本語レポートを生成する。

![コンセプト図: 生成AIの成果物ではなくプロンプトを見ることで、裏側にある意図や認識を指導できるという考え方を示した図。従来の成果物指導との違いを対比している。](image.png)

## 背景

生成AIの普及により、成果物の指導だけでは部下の意図や認識を正しく把握できなくなった。従来は成果物と意図・認識が不可分だったため、成果物を直すことが指導になっていたが、生成AIの出力をそのまま成果物として利用する場合、成果物を見ても裏側にある意図や認識は見えない。

**プロンプトには意図が埋まっている。** プロンプトを分析することで、その人が何を理解し、何を理解していないのかを推定できる。本ツールはこの考え方に基づき、AIとの対話履歴を自動収集・分析してレポートを生成する。

## 対応ツール

| ツール | ログ形式 |
|--------|---------|
| Codex | JSONL（`history.jsonl` + `sessions/YYYY/MM/DD/*.jsonl`） |
| Claude Code（CLI / VS Code拡張） | JSONL（history.jsonl + プロジェクト別セッション） |
| GitHub Copilot Chat | SQLite（state.vscdb） |
| Cline | JSON（api_conversation_history.json） |
| Roo Code | JSON（Clineと同一構造） |
| Windsurf (Cascade) | テキスト（自動要約メモリ） |
| Google Antigravity | テキスト（ログファイル） |

## 使い方

### Claude Code

Claude Code 上で `/prompt-review` を実行する。

```text
/prompt-review              # 全プロジェクト、過去7日分（デフォルト）
/prompt-review 30           # 過去30日分
/prompt-review yonshogen    # 特定プロジェクトのみ
/prompt-review yonshogen 30 # 特定プロジェクト × 過去30日分
/prompt-review 0            # 全期間
```

### Codex

Codex 上では `prompt-review` スキル名を含めて依頼する。

```text
$prompt-review
$prompt-review 30
$prompt-review yonshogen
$prompt-review yonshogen 30
$prompt-review 0
```

レポートは `reports/prompt-review-YYYY-MM-DD.md` に出力される。

## レポートの構成

1. **データソースサマリー** - 検出ツール・メッセージ数・期間
2. **プロジェクト別サマリー** - プロジェクトごとの作業概要
3. **技術理解度マップ** - 熟知 / 基本理解 / 学習中の3段階分類
4. **プロンプティング力の評価** - 強み・改善ポイント・特徴的な癖
5. **AI活用スタイル** - 主体的活用 / 依存傾向 / ツール別傾向
6. **成長の軌跡と学習提案** - 時系列変化・推奨ステップ
7. **総合評価サマリー**

※ シークレット警告セクションはクレデンシャル検出時のみ出力

## サンプルレポート

実際の出力例は [こちらのGist](https://gist.github.com/tokoroten/07398032d25a8f82f6452309ca402bff) を参照。

## ファイル構成

```
prompt-review/
├── README.md
├── reports/                              # 生成されたレポート
│   └── prompt-review-YYYY-MM-DD.md
├── .codex/
│   └── skills/
│       └── prompt-review/
│           ├── SKILL.md                  # Codex 用スキル定義
│           ├── agents/
│           │   └── openai.yaml           # Codex UIメタデータ
│           ├── scripts/
│           │   └── collect.py            # データ収集スクリプト
│           └── references/
│               ├── data-sources.md       # ログ保存場所・形式の詳細
│               └── report-template.md    # レポート構造テンプレート
└── .claude/
    └── skills/
        └── prompt-review/
            ├── SKILL.md                  # スキル定義（実行手順）
            ├── scripts/
            │   └── collect.py            # データ収集スクリプト
            └── references/
                ├── data-sources.md       # ログ保存場所・形式の詳細
                └── report-template.md    # レポート構造テンプレート
```

## 要件

- Claude Code（CLI または VS Code拡張）または Codex
- Python 3.10+
- SQLite3（GitHub Copilot Chat の解析に必要）
