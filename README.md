# claude-rc-reattach

Claude Code のアカウントを切り替えたあと、**切り替え前に Remote Control（RC）接続していた会話を、新しいアカウントの RC セッションとして復帰させる**ツールです。

## 背景 — なぜこれが必要か

Claude Code の Remote Control セッションは**サーバー側でアカウントに紐づいている**ため、アカウントを切り替えると旧アカウントの RC セッションは新アカウントから掴み直せません（claude.ai/code 上に非アクティブ表示で残り続けます）。

一方で**会話履歴そのものはローカル**（`~/.claude/projects/`）に保存されており、アカウントとは無関係です。このツールは以下の流れでその性質を利用します。

```
切り替え前                          切り替え後
─────────                          ─────────
save: RC接続中の会話を記録    →    restore: 各会話を claude --resume で
（~/.claude/sessions/ を走査）      tmux 内に立ち上げ直す
                                    → 新アカウント配下の新しい RC セッション
                                      として再登録される（会話の中身は継続）
```

RC 接続中かどうかは、Claude Code が実行中セッションごとに書き出す `~/.claude/sessions/<PID>.json` の `bridgeSessionId` フィールドの有無で判定しています。

## 動作要件

- macOS（Claude Code v2.1.220 で検証。Linux でも動く見込みですが未検証）
- Claude Code **v2.1.200 以降**
- `tmux`（`brew install tmux`）
- `python3`（Xcode Command Line Tools に付属）
- claude.ai の **Pro / Max / Team / Enterprise** プラン（RC の利用条件）
- 復帰対象の各ディレクトリで一度 `claude` を起動し、trust ダイアログを承認済みであること

### 推奨設定

`~/.claude/settings.json` に以下を追加しておくと、resume したセッションが自動的に RC 登録されます（このツールの前提設定です）:

```json
{
  "remoteControlAtStartup": true
}
```

> **注意**: この設定を入れても `/config` 画面の「Enable Remote Control for all sessions」が
> false 表示にリセットされて見える既知バグがあります（[#29929](https://github.com/anthropics/claude-code/issues/29929)）。
> 実際に有効かどうかは **/config の表示ではなく、セッションのフッターに `/rc active` が出るか**で確認してください。

## インストール

```bash
git clone https://github.com/igarashi-hobby/claude-rc-reattach.git
cd claude-rc-reattach
chmod +x bin/claude-rc-reattach

# PATH の通った場所にシンボリックリンクを作成
ln -s "$(pwd)/bin/claude-rc-reattach" ~/.local/bin/claude-rc-reattach
# （~/.local/bin が PATH にない場合は .zshrc に export PATH="$HOME/.local/bin:$PATH" を追加）
```

動作確認:

```bash
claude-rc-reattach list   # RC 接続中の会話が一覧表示されれば OK
```

## 使い方

### 一括実行（推奨）

```bash
claude-rc-reattach switch
```

以下を順に実行します:

1. **save** — RC 接続中の会話を manifest に保存
2. 既存セッションの終了を促す（開いたままだと同じ会話への多重書き込みになるため）
3. `claude auth logout` → `claude auth login`（新アカウントで対話ログイン）
4. **restore** — 各会話を tmux セッション `rc-restore` 内で resume

完了後、`tmux attach -t rc-restore` で各ウィンドウのフッターに `/rc active` が出ていれば成功です。新アカウントの claude.ai/code / モバイルアプリに新しいセッションとして表示されます。

### 個別実行

| コマンド | 実行タイミング | 動作 |
|---|---|---|
| `list` | いつでも | RC 接続中の会話を表示（save 対象の確認） |
| `save` | 切り替え**前** | RC 接続中の会話を manifest に保存 |
| `restore` | 切り替え**後**（ログイン済み） | manifest の会話を tmux で resume |

```bash
# 切り替え前
claude-rc-reattach save

# （ターミナルを閉じ、logout → login した後）
claude-rc-reattach restore
```

### オプション（restore / switch）

| オプション | 動作 |
|---|---|
| `--fork` | 元プロセスが開いたままの会話を `--fork-session` で複製として復帰（元の会話は変更しない） |
| `--dry-run` | 実際には起動せず、何が行われるかだけ表示 |

### 環境変数

| 変数 | 既定値 | 意味 |
|---|---|---|
| `CLAUDE_RC_MANIFEST` | `~/.local/state/claude-rc-reattach/manifest.tsv` | manifest の保存先 |
| `CLAUDE_RC_TMUX_SESSION` | `rc-restore` | 復帰先の tmux セッション名 |

## 制約・知っておくべきこと

- **RC セッション ID は新規になります。** claude.ai/code 上では「新しいセッション」として表示されます（会話の中身は継続）。旧アカウント側のセッション一覧には非アクティブなエントリが残りますが、これは仕様上消せません
- **`claude auth login` だけは対話操作が必須**です。環境変数やトークンで代替する手段はありません（`claude setup-token` のトークンでは RC を張れないことが公式ドキュメントに明記されています）
- 同じ会話を別プロセスで開いたまま restore すると transcript の多重書き込みになるため、既定では skip します。複製でよければ `--fork` を使ってください
- `save` は **RC 接続中のセッションが生きている間に**実行する必要があります。セッションを閉じると `~/.claude/sessions/` の状態ファイルが消え、検出できなくなります
- `~/.claude/sessions/<PID>.json` は非公開の内部仕様です（v2.1.220 で動作検証）。Claude Code のアップデートで形式が変わる可能性があります。動かなくなったら `claude-rc-reattach list` の結果と `ls ~/.claude/sessions/` を見比べてください

## トラブルシューティング

| 症状 | 確認すること |
|---|---|
| `list` に何も出ない | 各セッションのフッターに `/rc active` が出ているか。出ていなければそのセッションは RC 未接続（`/remote-control` で手動接続するか、`remoteControlAtStartup: true` を設定して開き直す） |
| restore 後に `/rc active` が出ない | `claude doctor` で RC の適格性チェックを確認。`ANTHROPIC_API_KEY` が環境にセットされていると RC が無効になるため `unset` する |
| ウィンドウがすぐ落ちる | tmux 3.2 以降なら異常終了した pane が残るので、そこにエラーが表示されています。`tmux attach -t rc-restore` で確認 |
| 「会話ログが見つかりません」で skip される | 該当プロジェクトの `~/.claude/projects/<エンコード名>/<sessionId>.jsonl` が存在するか確認 |

## ライセンス

MIT
