# データソース詳細リファレンス

各AIツールのログ保存場所・フォーマット・抽出方法の詳細。

---

## 1. Codex

### 保存場所
| OS | パス |
|----|------|
| Windows | `%USERPROFILE%\\.codex\\` |
| macOS | `~/.codex/` |
| Linux | `~/.codex/` |

### ファイル構造
```text
~/.codex/
├── history.jsonl                         # 全セッション横断の軽量インデックス
└── sessions/
    └── YYYY/MM/DD/
        └── rollout-*.jsonl              # セッション別の完全なイベントログ
```

### history.jsonl の形式
```json
{"session_id":"...","ts":1770877327,"text":"ユーザーの入力テキスト"}
```

### セッションJSONL の形式
各行がJSON。ユーザーメッセージは `response_item` かつ `payload.type: "message"` / `payload.role: "user"`。
```json
{"type":"session_meta","payload":{"id":"...","cwd":"/path/to/project"}}
{"type":"response_item","timestamp":"2026-03-12T09:44:04.430Z","payload":{"type":"message","role":"user","content":[{"type":"input_text","text":"実装に進んでください。"}]}}
```

### 抽出方法（2ソース方式）

**ソース1: `sessions/YYYY/MM/DD/*.jsonl`（主要ソース）**
1. `session_meta` から `cwd` と `session id` を取得
2. `response_item` のうち `role: "user"` のメッセージだけを抽出
3. `content[].type == "input_text"` のテキストを連結
4. `cwd` の末尾ディレクトリ名をプロジェクト名として採用

**ソース2: `history.jsonl`（補完ソース）**
1. `text` と `ts` を読み込む
2. `session_id` をキーにソース1で得た `cwd` / プロジェクト名を補完する
3. 同一 `session_id + timestamp + text` は重複排除する

### 注意事項
- `history.jsonl` の `ts` は epoch 秒で保存されるため、ミリ秒へ正規化して扱う
- セッションJSONLには `session_meta` や tool call も含まれるため、ユーザー発話のみを抽出する
- Codex のセッションログは `cwd` を持つので、プロジェクトフィルタはディレクトリ名とフルパス双方で判定できる

---

## 2. Claude Code（主要ソース）

### 保存場所
| OS | パス |
|----|------|
| Windows | `C:\Users\<user>\.claude\` |
| macOS | `~/.claude/` |
| Linux | `~/.claude/` |

### ファイル構造
```
~/.claude/
├── history.jsonl                          # 全プロジェクトの軽量インデックス
└── projects/
    └── {encoded-path}/                    # パス区切り(\, /, :) → - に変換
        ├── {session-uuid}.jsonl           # セッション別の完全な会話ログ
        └── {session-uuid}/
            └── subagents/                 # サブエージェントのログ
```

### history.jsonl の形式
```json
{"display":"ユーザーの入力テキスト","pastedContents":{},"timestamp":1759115795246,"project":"D:\\path\\to\\project"}
```

### プロジェクト別JSONL の形式
各行がJSON。ユーザーメッセージは `"type":"user"` を含む。
```json
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"..."}]},"timestamp":"..."}
```

### 抽出方法（2ソース方式）

**ソース1: history.jsonl（CLI使用時）**
1. `history.jsonl` を読み込み、`display` フィールドからプロンプトを取得
2. `/clear` やファイルパスのみの行を除外
3. 収集済みの `sessionId` を記録（ソース2との重複排除用）

**ソース2: プロジェクト別セッションJSONL（VS Code拡張機能使用時）**
1. `projects/` 配下の各プロジェクトディレクトリを走査
2. 各ディレクトリの `*.jsonl` ファイルから `"type":"user"` のメッセージを抽出
3. `isMeta: true` のシステムメッセージはスキップ
4. `<ide_opened_file>`, `<local-command-stdout>` 等のシステムタグを除外
5. history.jsonl で収集済みのセッションIDはスキップ（重複排除）
6. 各プロジェクト最新5ファイル、1ファイルあたりユーザーメッセージ100件上限

**重要**: VS Code拡張機能で使用した場合、`history.jsonl` にはエントリが記録されず、
プロジェクト別セッションJSONLにのみ会話ログが保存される。両方のソースを読む必要がある。

### パスエンコード規則
プロジェクトディレクトリ名は、元のパスの `\`, `/`, `:` を `-` に変換。
例: `C:\Users\shinta\Documents\GitHub\yonshogen` → `c--Users-shinta-Documents-GitHub-yonshogen`

---

## 3. GitHub Copilot Chat

### 保存場所
| OS | パス |
|----|------|
| Windows | `%APPDATA%\Code\User\workspaceStorage\*\state.vscdb` |
| macOS | `~/Library/Application Support/Code/User/workspaceStorage/*/state.vscdb` |
| Linux | `~/.config/Code/User/workspaceStorage/*/state.vscdb` |

### ファイル形式
SQLite データベース（`state.vscdb`）

### 抽出方法
```bash
# 利用可能なキーの一覧を取得
sqlite3 "<path>/state.vscdb" "SELECT key FROM ItemTable WHERE key LIKE '%chat%';"

# チャットセッションインデックスを取得
sqlite3 "<path>/state.vscdb" "SELECT value FROM ItemTable WHERE key = 'interactive.sessions';" 2>/dev/null

# 代替キー（バージョンにより異なる）
sqlite3 "<path>/state.vscdb" "SELECT value FROM ItemTable WHERE key = 'chat.ChatSessionStore.index';" 2>/dev/null
```

### 注意事項
- `sqlite3` コマンドが必要（Windows では Git Bash 付属のものや別途インストール）
- ワークスペースごとに別の `state.vscdb` が存在する
- セッションデータはJSON文字列としてvalueカラムに格納
- ユーザーのプロンプトは `request` や `message` フィールドに含まれる

---

## 4. Cline

### 保存場所
| OS | パス |
|----|------|
| Windows | `%APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\` |
| macOS | `~/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/` |
| Linux | `~/.config/Code/User/globalStorage/saoudrizwan.claude-dev/` |

### ファイル構造
```
saoudrizwan.claude-dev/
├── state/
│   └── taskHistory.json              # タスク履歴インデックス
└── tasks/
    └── {task-id}/
        ├── api_conversation_history.json  # API会話ログ
        ├── ui_messages.json               # UI表示メッセージ
        └── task_metadata.json             # タスクメタデータ
```

### 抽出方法
1. `taskHistory.json` を読み込んでタスク一覧を取得
2. 各タスクの `api_conversation_history.json` から `role: "human"` のメッセージを抽出
3. サンプリング: 最新20タスクに制限

---

## 5. Roo Code

### 保存場所
| OS | パス |
|----|------|
| Windows | `%APPDATA%\Code\User\globalStorage\RooVeterinaryInc.roo-cline\` |
| macOS | `~/Library/Application Support/Code/User/globalStorage/RooVeterinaryInc.roo-cline/` |
| Linux | `~/.config/Code/User/globalStorage/RooVeterinaryInc.roo-cline/` |

### ファイル構造
Cline と同一構造（Roo Code は Cline のフォーク）。
```
RooVeterinaryInc.roo-cline/
├── state/
│   └── taskHistory.json
└── tasks/
    └── {task-id}/
        ├── api_conversation_history.json
        ├── ui_messages.json
        └── task_metadata.json
```

### 抽出方法
Cline と同じ手順。

---

## 6. Windsurf (Cascade)

### 保存場所
| OS | パス |
|----|------|
| Windows | `%USERPROFILE%\.codeium\windsurf\memories\` |
| macOS | `~/.codeium/windsurf/memories/` |
| Linux | `~/.codeium/windsurf/memories/` |

バックアップ（cascade-backup-utils 使用時）:
`~/.cascade_backups/`

### ファイル形式
メモリファイル（テキスト形式）。会話の直接ログではなく、Cascadeが自動生成した要約・メモリ。

### 抽出方法
1. `~/.codeium/windsurf/memories/` 配下をGlobで検索
2. テキストファイルをReadで読み込み
3. `~/.cascade_backups/` が存在する場合はそちらも読み込み
4. メモリファイルのため、直接のプロンプトではなくCascadeの要約情報として扱う

### 注意事項
- Windsurf の会話ログ自体はローカルに直接保存されない場合がある
- memories/ はワークスペース単位で分離されている
- メモリの内容は要約であり、元のプロンプトとは異なる

---

## 7. Google Antigravity

### 保存場所
| OS | パス |
|----|------|
| Windows | `%USERPROFILE%\.gemini\antigravity\brain\` |
| macOS | `~/.gemini/antigravity/brain/` + `~/.gemini/antigravity/conversations/` |
| Linux | `~/.gemini/antigravity/brain/` |

### ファイル構造
```
~/.gemini/antigravity/
├── brain/
│   └── {conversation-id}/
│       └── .system_generated/
│           └── logs/                  # 会話ログ
└── conversations/
    └── *.pb                           # Protocol Buffers 形式
```

### 抽出方法
1. `~/.gemini/antigravity/brain/` 配下をGlobで探索
2. `.system_generated/logs/` 内のテキストファイルを読み込み
3. `.pb` ファイル（Protocol Buffers）はバイナリのため直接読み取り不可 → スキップ
4. テキスト形式のログファイルのみ対象

### 注意事項
- Antigravity は比較的新しいツールのため、ログ形式が変更される可能性がある
- `.gemini/` フォルダが削除されると会話リストは残るが内容は読めなくなる（既知のバグ）
- Protocol Buffers 形式のファイルはテキストとして読めないためスキップする

---

## 共通のサンプリング戦略

大量のログデータを効率的に処理するため、以下の戦略を適用する:

| パラメータ | 値 | 理由 |
|-----------|-----|------|
| プロジェクトあたり最大ファイル数 | 5 | コンテキストウィンドウの制約 |
| ファイルあたり最大ユーザーメッセージ数 | 100 | 十分な分析量を確保しつつ制限 |
| Cline/Roo Code 最大タスク数 | 20 | 直近の活動に焦点 |
| 日数フィルタ | 引数で指定 | timestamp を現在時刻と比較 |

### 日数フィルタの適用方法
- Codex: セッションJSONL内の `timestamp`、history.jsonl の `ts` を現在時刻と比較
- Claude Code: `timestamp` フィールド（Unix epoch ミリ秒）で比較
- Cline/Roo Code: `task_metadata.json` のタイムスタンプで比較
- GitHub Copilot Chat: セッションデータ内のタイムスタンプで比較
- Windsurf/Antigravity: ファイルの更新日時で比較（正確なタイムスタンプが取れない場合）

### 存在チェックの順序
1. まず各ツールのベースディレクトリが存在するかをGlobまたはlsで確認
2. 存在するツールのみデータ収集を実行
3. 検出されなかったツールはレポートの「データソースサマリー」に「未検出」と記載
