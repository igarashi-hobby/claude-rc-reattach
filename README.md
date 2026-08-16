# claude-rc-reattach

Claude Code のアカウントを切り替えると、Remote Control（RC）接続中だった会話は新しいアカウントから操作できなくなります。
このツールは、その会話を**新しいアカウントの RC セッションとして復帰**させます。
会話の中身はそのまま引き継がれます。

## 使い方は 3 ステップ

```bash
# ① 切り替え前: RC 接続中の会話を記録する
claude-rc-reattach save

# ② アカウントを切り替える（会話を開いているターミナルは閉じておく）
claude auth logout
claude auth login

# ③ 切り替え後: 記録した会話を復帰させる
claude-rc-reattach restore
```

restore は起動後に各会話の RC 登録を自動確認し（既定で最大 60 秒）、会話ごとに ✔ / ⚠ を表示します。
tmux ウィンドウの起動成功と RC 登録の成功は別物のため、bridgeSessionId の書き込みを実測で確認しています。
⚠ が残った場合は次で再接続を試せます:

```bash
claude-rc-reattach reconnect
```

手動で確認したい場合:

```bash
tmux attach -t rc-restore
# 各ウィンドウのフッターに「/rc active」が出ていれば成功。
# 新アカウントの claude.ai/code やモバイルアプリに新しいセッションとして表示されます。
```

①〜③ を対話形式で一括実行する `claude-rc-reattach switch` もあります（後述）。

## インストール

### 必要なもの

| 要件 | 補足 |
|---|---|
| macOS | Claude Code v2.1.232 で検証（Linux は動く見込みだが未検証） |
| Claude Code v2.1.200 以降 | |
| tmux | `brew install tmux`。restore / switch / reconnect で必要（無い場合は実行時にインストール方法を案内して停止します。list / save は tmux 不要） |
| python3 | Xcode Command Line Tools に付属 |
| Pro / Max / Team / Enterprise プラン | Remote Control の利用条件 |

### 手順

```bash
git clone https://github.com/igarashi-hobby/claude-rc-reattach.git
cd claude-rc-reattach
chmod +x bin/claude-rc-reattach

# ここから先はリポジトリのルート（cd した直後のディレクトリ）で実行すること
# PATH の通った場所にシンボリックリンクを作成
mkdir -p ~/.local/bin
ln -sf "$PWD/bin/claude-rc-reattach" ~/.local/bin/claude-rc-reattach

# リンク先が実在するか確認（No such file と出たら別ディレクトリで実行している）
ls -L ~/.local/bin/claude-rc-reattach

# ~/.local/bin が PATH にない場合は .zshrc に以下を追加:
#   export PATH="$HOME/.local/bin:$PATH"
```

動作確認:

```bash
claude-rc-reattach list   # RC 接続中の会話が一覧表示されれば OK
```

### 推奨設定

`~/.claude/settings.json` に以下を追加してください。
resume したセッションが自動的に RC 登録されるようになります（このツールの前提設定です）:

```json
{
  "remoteControlAtStartup": true
}
```

> **注意**: この設定を入れても `/config` 画面の「Enable Remote Control for all sessions」が false 表示にリセットされて見える既知バグがあります（[#29929](https://github.com/anthropics/claude-code/issues/29929)）。
> 有効かどうかは **/config の表示ではなく、セッションのフッターに `/rc active` が出るか**で確認してください。

## コマンド

| コマンド | 実行タイミング | 動作 |
|---|---|---|
| `list` | いつでも | RC 接続中の会話を一覧表示（save 対象の事前確認用） |
| `save` | 切り替え**前** | RC 接続中の会話を manifest ファイルに記録 |
| `restore` | 切り替え**後**（ログイン済み） | manifest の各会話を tmux 内で resume し、RC 再登録と登録確認まで行う |
| `switch` | — | save → logout → login → restore を対話形式で一括実行 |
| `reconnect` | restore 後いつでも | tmux 内の RC 未登録セッションに `/remote-control` を送信して再接続 |

### switch（一括実行）

```bash
claude-rc-reattach switch
```

以下を順に実行します。
途中で手を動かすのは「セッションを閉じる」「新アカウントでログインする」の 2 箇所だけです。

1. **save** — RC 接続中の会話を manifest に記録
2. **セッション終了の確認** — 会話を開いているターミナル / クライアントを閉じて Enter
   （Enter 時に閉じ忘れを自動検知し、まだ開いている会話があれば列挙して再確認します。
   開いたままでも復帰はできますが、その会話は「複製」として復帰します。詳細は下記）
3. **アカウント切り替え** — `claude auth logout` 後、新アカウントで対話ログイン
4. **restore** — 各会話を tmux セッション `rc-restore` 内で resume し、RC 登録を自動確認

### オプション（restore / switch 共通）

| オプション | 動作 |
|---|---|
| `--no-fork` | 元プロセスが開いたままの会話を複製せず skip する |
| `--no-inherit-perms` | save 時に検出した権限モードを引き継がない |
| `--dry-run` | 実際には起動せず、何が行われるかだけ表示 |

**元プロセスが開いたままの会話の扱い（既定 = 複製復帰）**:
同じ会話を 2 つのプロセスで同時に開くと会話ログへの多重書き込みが起きるため、開いたままの会話は既定で `--fork-session` を付けて「複製」として復帰します。
元の会話は変更されません。
ただし複製後は 2 つの会話が別々に進むため、**同一の会話として復帰させたい場合は、restore の前に元のターミナルを閉じてください**。
複製も作りたくない場合は `--no-fork` を付けると該当の会話を skip します。

**古い会話は復帰対象外（既定 = 最終チャットから 3 日以内のみ）**:
会話ログの最終エントリが 3 日より前の会話は、放置されたものとみなして restore で skip します。
上限日数は環境変数 `CLAUDE_RC_MAX_AGE_DAYS` で変更でき、`0` にするとこのチェック自体を無効化できます。

### RC 登録の自動確認と reconnect

tmux ウィンドウの起動成功と RC 登録の成功は別物です。
RC 登録は resume されたプロセスが起動後に非同期で行うため、restore は各セッションの状態ファイル（`~/.claude/sessions/<PID>.json`）に `bridgeSessionId` が書き込まれるのを実測で確認してから ✔ を表示します（既定で最大 60 秒。`CLAUDE_RC_VERIFY_TIMEOUT` で変更、`0` で無効化）。
claude が異常終了した場合は pane を残し（tmux 3.2 以降）、その旨を表示します。

また、復帰直後は `/rc active` でも、**しばらく経ってから credential の更新失敗で RC だけが切断されることがあります**（下記トラブルシューティング参照）。
その場合は `reconnect` で一括再接続できます:

```bash
claude-rc-reattach reconnect
# tmux セッション rc-restore 内の各ウィンドウの RC 状態を調べ、
# 未登録のものにだけ /remote-control を送信 → 登録を再確認します
```

reconnect の対象は tmux セッション（既定: `rc-restore`）内の pane のみです。
通常のターミナルで開いているセッションにはキー送信ができないため、そちらは手動で `/remote-control` を実行してください。
また、対象ウィンドウの入力欄に入力中の文字が残っていると送信内容が混ざるため、入力欄は空にしておいてください。

restore はセッションの同時起動による credential の同時発行・同時失効を緩和するため、各ウィンドウの起動間隔を既定で 3 秒空けます（`CLAUDE_RC_STAGGER_SECS` で変更、`0` で無効化）。

### 権限モードの引き継ぎ（--dangerously-skip-permissions 等）

`--dangerously-skip-permissions` などの権限モードは、会話ではなく**プロセス起動時の状態**のため、`--resume` では引き継がれません。
そこで save の時点で元プロセスの起動引数を読み、restore で自動的に付け直します（既定で有効）。

```
$ claude-rc-reattach save
✔ 2 件の会話を保存しました → ...
  • my-project  [/path/to/project]  権限モード: --dangerously-skip-permissions

$ claude-rc-reattach restore
  my-project: 権限モードを引き継ぎます（--dangerously-skip-permissions）
```

引き継ぎ対象は `--dangerously-skip-permissions` / `--allow-dangerously-skip-permissions` / `--permission-mode` / `--permission-prompt-tool` です。
`--model` などそれ以外のオプションは引き継ぎません。

**有効にした方法によって挙動が変わります**:

| bypass を有効にした方法 | 復帰後 |
|---|---|
| `claude --dangerously-skip-permissions` で起動 | **自動で引き継がれる**（save 時に検出） |
| `~/.claude/settings.json` の `defaultMode` | 元から適用される（このツールは何もしない） |
| セッション中に `/permissions` で変更 | 引き継がれない（どこにも記録されないため検出不可） |

引き継ぎたくない場合は `--no-inherit-perms` を付けてください。
手動で指定したい場合や、検出できないケースを補いたい場合は、環境変数 `CLAUDE_RC_CLAUDE_ARGS` が使えます。
restore 時の `claude` コマンドにそのまま追加され、**引き継ぎ対象のフラグ（上記 4 種のいずれか）を指定した場合は自動引き継ぎより優先されます**（二重指定にはなりません）:

```bash
CLAUDE_RC_CLAUDE_ARGS="--dangerously-skip-permissions" claude-rc-reattach restore
# switch でも同様に有効
CLAUDE_RC_CLAUDE_ARGS="--dangerously-skip-permissions" claude-rc-reattach switch

# 複数指定・値にスペースを含む場合（内側は '...' で囲む）
CLAUDE_RC_CLAUDE_ARGS="--dangerously-skip-permissions --append-system-prompt 'よろしく'" \
  claude-rc-reattach restore
```

このツールに関係なく、すべてのセッションを常に bypass permissions で起動したい場合は、`~/.claude/settings.json` に恒久設定する方法もあります。
この場合は復帰後のセッションにも自動的に効くため、上記の引き継ぎは不要です:

```json
{
  "permissions": { "defaultMode": "bypassPermissions" }
}
```

### 環境変数

いずれも**設定は任意**です（既定値のままで動作します）。

| 変数 | 既定値 | 意味 |
|---|---|---|
| `CLAUDE_RC_MANIFEST` | `~/.local/state/claude-rc-reattach/manifest.tsv` | manifest の保存先 |
| `CLAUDE_RC_TMUX_SESSION` | `rc-restore` | 復帰先の tmux セッション名 |
| `CLAUDE_RC_CLAUDE_ARGS` | （なし） | restore 時に `claude` コマンドへ追加するオプション（スペース区切りで複数可。値にスペースを含む場合は `'...'` か `"..."` で囲む） |
| `CLAUDE_RC_MAX_AGE_DAYS` | `3` | 最終チャットからこの日数を超えた会話は restore で skip する（`0` で無効化） |
| `CLAUDE_RC_VERIFY_TIMEOUT` | `60` | restore / reconnect 後に RC 登録を確認する最大秒数（`0` で確認しない） |
| `CLAUDE_RC_STAGGER_SECS` | `3` | restore で各セッションを起動する間隔の秒数（credential の同時発行・同時失効の緩和用。`0` で無効化） |

## 仕組み

RC セッションは**サーバー側でアカウントに紐づいている**ため、アカウントを切り替えると旧アカウントの RC セッションは掴み直せません。
一方、**会話履歴そのものはローカル**（`~/.claude/projects/`）に保存されており、アカウントとは無関係です。
このツールはその性質を利用します。

```
切り替え前                          切り替え後
─────────                          ─────────
save: RC接続中の会話を記録    →    restore: 各会話を claude --resume で
（~/.claude/sessions/ を走査）      tmux 内に立ち上げ直す
                                    → 新アカウント配下の新しい RC セッション
                                      として再登録される（会話の中身は継続）
```

RC 接続中かどうかは、Claude Code が実行中セッションごとに書き出す `~/.claude/sessions/<PID>.json` の `bridgeSessionId` フィールドの有無で判定しています。

権限モードはこの状態ファイルには含まれないため、save 時に同ファイルの `pid` から `ps` で起動引数を読み、権限関連のフラグだけを manifest に記録しています。

## 制約・知っておくべきこと

- **RC セッション ID は新規になります。**
  claude.ai/code 上では「新しいセッション」として表示されます（会話の中身は継続）。
  旧アカウント側のセッション一覧には非アクティブなエントリが残りますが、これは仕様上消せません
- **復帰後も RC が切断されることがあります。**
  RC は接続後も短命の credential を定期更新しており、更新失敗（トークン失効・並行セッションの refresh 競合など）で切断されることがあります。
  restore の確認が ✔ でも後から切断され得るため、切れていたら `reconnect` を実行してください
- **デスクトップ版 / IDE 拡張のセッションも save の検出対象になりますが、復帰先は常に tmux 内の CLI セッションです。**
  検出はセッション状態ファイルの `kind: "interactive"` で判定しており、起動元（entrypoint）は問いません。
  復帰した会話をデスクトップ版で続けたい場合は、復帰後にデスクトップ側で同じ会話を開き直してください
- **`claude auth login` だけは対話操作が必須**です。
  環境変数やトークンで代替する手段はありません（`claude setup-token` のトークンでは RC を張れないことが公式ドキュメントに明記されています）
- **複製（fork）復帰した会話は元の会話と分岐します。**
  復帰後に元のターミナル側で会話を続けても、復帰した側には反映されません
- **`save` は RC 接続中のセッションが生きている間に実行してください。**
  セッションを閉じると `~/.claude/sessions/` の状態ファイルが消え、検出できなくなります。
  権限モードも生きているプロセスの起動引数から読むため、同じタイミングでしか取得できません
- **権限モードの検出は `ps` の起動引数から行うため、引用符の情報は失われます。**
  `--append-system-prompt "... --permission-mode plan ..."` のように他オプションの値の中に権限フラグと同じ語が含まれていると、誤って権限モードとして検出することがあります。
  その場合は `--no-inherit-perms` で引き継ぎを無効化してください
- `~/.claude/sessions/<PID>.json` は非公開の内部仕様です（v2.1.232 で動作検証）。
  Claude Code のアップデートで形式が変わる可能性があります。
  動かなくなったら `claude-rc-reattach list` の結果と `ls ~/.claude/sessions/` を見比べてください

## トラブルシューティング

| 症状 | 確認すること |
|---|---|
| `list` に何も出ない | 各セッションのフッターに `/rc active` が出ているか。出ていなければそのセッションは RC 未接続（`/remote-control` で手動接続するか、`remoteControlAtStartup: true` を設定して開き直す） |
| restore 後に `/rc active` が出ない | restore の RC 登録確認で ⚠ になった会話は `claude-rc-reattach reconnect` で再接続を試す。それでもだめなら `claude doctor` で RC の適格性チェックを確認。`ANTHROPIC_API_KEY` が環境にセットされていると RC が無効になるため `unset` する |
| 復帰後しばらくして `Remote Control disconnected — Transport closed: worker credential expired or rejected (code 4094)` や `could not fetch fresh session credentials after code 401` が出る | RC の worker credential 更新に失敗して切断された状態。旧アカウントのまま開き続けたセッションでは logout によるトークン失効のため必ず発生する（そのセッションの RC は復旧不可。新アカウントで開き直す）。同一アカウントでも複数セッション並行時の OAuth refresh 競合で発生することがある Claude Code 側の既知問題（[#54443](https://github.com/anthropics/claude-code/issues/54443) / [#61912](https://github.com/anthropics/claude-code/issues/61912) / [#78309](https://github.com/anthropics/claude-code/issues/78309)）。tmux 内のセッションなら `claude-rc-reattach reconnect`、通常のターミナルなら該当セッションで `/remote-control` を実行して再接続する |
| ウィンドウがすぐ落ちる | tmux 3.2 以降なら異常終了した pane が残るので、そこにエラーが表示されています。`tmux attach -t rc-restore` で確認 |
| 「会話ログが見つかりません」で skip される | 該当プロジェクトの `~/.claude/projects/<エンコード名>/<sessionId>.jsonl` が存在するか確認 |
| 「別プロセスで開いたままです」で skip される | `--no-fork` を付けた場合の動作です。外せば複製として復帰します（既定） |
| 「最終チャットが N 日前」で skip される | 既定で最終チャットが 3 日より古い会話は復帰しません。`CLAUDE_RC_MAX_AGE_DAYS` で上限を変更するか、`CLAUDE_RC_MAX_AGE_DAYS=0` で無効化できます |
| 復帰後に `--dangerously-skip-permissions` が外れている | save 時の manifest 5 列目に記録されているか確認（`save` の出力に「権限モード:」が出ていれば検出済み）。セッション中に `/permissions` で変更した場合は検出できないため、`CLAUDE_RC_CLAUDE_ARGS` で明示指定してください（上記「権限モードの引き継ぎ」参照） |

## ライセンス

MIT
