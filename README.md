# claude-rc-reattach

Claude Code のアカウントを切り替えると、Remote Control（RC）接続中だった会話は新しいアカウントから操作できなくなります。
このツールは、その会話を**新しいアカウントの RC セッションとして復帰**させます。
ターミナル内の会話（文脈）はそのまま引き継がれます。

> **対象は Claude Code CLI のセッションだけです。**
> Claude Desktop アプリは CLI とは別に認証を持つため、このツールでは切り替わりません（アプリ側で個別にサインインし直してください）。
> また、**復帰後のスマホ / claude.ai/code には切り替え前のやりとりは表示されません**（Claude Code 側の仕様。[詳細](#リモート側に切り替え前の履歴は表示されない)）。

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
| macOS | Claude Code v2.1.233 で検証（Linux は動く見込みだが未検証） |
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

### 推奨設定（任意）

restore は `claude --resume <sessionId> --remote-control <会話名>` の形で起動するため、**復帰させるだけならこの設定は不要**です。
普段のセッションも自動で RC 接続したい場合は、`~/.claude/settings.json` に以下を追加してください:

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
| `save` | 切り替え**前** | RC 接続中の会話を manifest ファイルに記録（一時ファイル → `mv` の原子的置換。ディレクトリ作成・書き込みに失敗した場合は「保存しました」と言わずエラーで終了し、既存の manifest も壊しません。switch はこの失敗で logout に進みません） |
| `restore` | 切り替え**後**（ログイン済み） | manifest の各会話を tmux 内で resume し、RC 再登録と登録確認まで行う |
| `switch` | — | save → logout → login → restore を対話形式で一括実行 |
| `recover` | save し損ねた時 | manifest を使わず、直近に更新された会話ログから復旧候補を探して resume |
| `prune` | 復帰後いつでも | 復元用 tmux セッション内で、会話ログの最終更新が古い pane を正常終了して片付ける |
| `reconnect` | restore 後いつでも | tmux 内の RC 未登録セッションに `/remote-control` を送信して再接続（`--recap` 可） |

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
| `--force` | すでに動いている会話も復帰させる（既定は skip） |
| `--no-fork` | `--force` 時、元プロセスが開いたままの会話を複製せず skip する |
| `--no-rename` | 復帰セッションに `-restored-MMDD` の名前を付けない |
| `--no-inherit-perms` | save 時に検出した権限モードを引き継がない |
| `--inherit-bypass` | bypass 系の権限モードも引き継ぐ（既定では引き継がない。後述） |
| `--recap` | RC 登録できた会話に要約依頼を 1 通送る（既定） |
| `--no-recap` | 要約依頼を送らない |
| `--limit <N>` | 起動する復元候補の上限（既定: `CLAUDE_RC_MAX_SESSIONS` / 15。`0` で無制限） |
| `--pick` | 復元候補を番号付き一覧から対話選択する（非対話環境ではエラー） |
| `--wave-size <N>` | `N` 件ずつ起動し、各波の RC 登録確認後に次の波へ進む（既定: `CLAUDE_RC_WAVE_SIZE` / 3。`0` で従来どおり一括） |
| `--resume-mode as-is\|summary` | 「Resume full session as-is」ダイアログの選択（既定: `as-is`） |
| `--yes` | メモリ予算と波ごとの RC 未登録確認プロンプトを省略 |
| `--dry-run` | 実際には起動せず、何が行われるかだけ表示（`switch` では manifest の上書きも logout / login も行いません） |

**終了コード（restore / switch）**:

| コード | 意味 |
|---|---|
| `0` | すべての会話を復帰し、RC 登録も確認できた |
| `1` | 1 件も復帰できなかった、または RC 登録を確認できなかった |
| `2` | 一部の会話を skip した（部分成功） |

**すでに動いている会話は復帰させない（重複ガード / 既定）**:
restore を 2 回実行したり、一部の会話を手で開き直した後に restore すると、同じ会話がいくつも立ち上がってしまいます。
これを防ぐため、restore は各会話について次の 2 つを調べ、どちらかに当てはまる会話を `skip (already running)` として飛ばします。

1. `~/.claude/sessions/*.json` のうち **PID が生きている**ものの `sessionId` が一致する（その会話が開かれている）
2. 実行中プロセスのコマンドラインに `--resume <対象の sessionId>` がある（その会話の resume / 複製がすでに動いている）

2 は複製復帰した会話を拾うためのものです（`--fork-session` で復帰すると走っているプロセス自身の sessionId は別物になり、1 だけでは検出できません）。

```
$ claude-rc-reattach restore
  skip (already running): my-project
```

**開いたままの会話も強制的に復帰させたい場合（`--force`）**:
`--force` を付けると従来どおり復帰します。
同じ会話を 2 つのプロセスで同時に開くと会話ログへの多重書き込みが起きるため、この場合は `--fork-session` を付けた「複製」として復帰します（元の会話は変更されません）。
複製後は 2 つの会話が別々に進むため、**同一の会話として復帰させたい場合は、restore の前に元のターミナルを閉じてください**。
複製も作りたくない場合は `--force --no-fork` で該当の会話を skip します。

**復帰セッションには名前が付きます（既定）**:
復帰した会話は `claude -n "<復元名の先頭 30 文字>-restored-MMDD"` で起動するため、claude.ai/code の一覧で元の会話と区別できます（例: `あすまるくん改修-immutable-cre-restored-0815`）。
元の名前が空、UUID 形式、または Claude の自動名（英数字とハイフンのみで `-MMDD-HHMM` 風の末尾）に見える場合は、会話ログから名前を作ります。
会話に付いている名前（`custom-title` / `agent-name`）があればそれを優先し、無ければ最初のユーザー発言の先頭 30 文字（改行・記号を除去）を使います。
Claude Code がスラッシュコマンド実行時などに先頭へ差し込む `<local-command-caveat>` / `<command-name>` / `<system-reminder>` だけの発言は名前に使わず、次の発言を見ます（素で採用すると `localcommandcaveatCaveat The m-restored-MMDD` のような名前になるため）。
それも取れない場合は cwd のディレクトリ名にフォールバックします。
既存名に `-restored-` が含まれる場合は、最初の `-restored-` 以降を落としてから当日の `-restored-MMDD` を付け直すため、復帰を重ねても名前が積み上がりません。
`-n` に未対応の古い claude では、その旨を表示して名前なしで起動します。
名前を付けたくない場合は `--no-rename` を付けてください。
tmux のウィンドウ名は `<sessionId の先頭 8 桁>-<会話名>`（会話名は 20 文字まで）です。
日本語はそのまま残します（非英数字を一括で `-` に潰すと「ミチガエル経営ダッシュボード」が `--------------` になり、attach しても会話を区別できず、手動 `kill-window` の誤爆や `reconnect` の出力での取り違えを招くため）。
tmux のターゲット指定で意味を持つ `.` `:` と空白・制御文字だけを `-` に置き換えます。

**復元候補は新しい順に上限付きで起動します**:
restore / recover は会話ログ（`*.jsonl`）の最終更新が新しい順に候補を並べ、既定では先頭 15 件だけ起動します。
上限は `--limit <N>` または `CLAUDE_RC_MAX_SESSIONS` で変更できます（`--limit 0` は無制限）。
上限を超えた候補がある場合は、未復元の件数と名前を表示し、追加で起動したいときの `claude-rc-reattach restore --limit ...` を案内します。
上限のカウントは「実際に起動する候補」だけを対象にします（restore は limit を掛ける前に、すでに動いている会話・復帰済みの会話・manifest 内の重複・作業ディレクトリや会話ログが無い行を除外します）。
実行中の会話は会話ログが最も新しいため先頭に並びます。これを数えてしまうと `--limit` の枠が skip 行で埋まり、案内どおり再実行しても未復元の会話に到達できないためです。

**復元候補を選んで起動できます**:
restore / recover に `--pick` を付けると、復元候補を「番号 / 最終更新 / 名前 / フォルダ」の一覧で表示し、`1,3,5` のようなカンマ区切りで起動対象を選べます。
`a` は全部、`q` は中止です。
`--limit` と併用した場合は、一覧自体が上限件数に絞られます。
標準入力が tty ではない非対話環境では `--pick` はエラーになります。

**起動前にメモリ予算を確認します**:
restore / recover は起動予定件数を 1 件あたり 600MB とみなし、macOS の `vm_stat` から free + inactive ページを空きメモリとして概算します。
予算が空きメモリの 70% を超える場合は、推奨 `--limit` を表示して `y/N` 確認で止めます。
問題ないと分かっている場合は `--yes` で確認を省略できます。

**Resume ダイアログの選択を指定できます**:
起動後の pane に `Resume full session as-is` を含むダイアログが出た場合、`--resume-mode as-is` ではその選択肢の番号を、`--resume-mode summary` では as-is ではない最初の番号選択肢を送って Enter します。
trust 確認（`Yes, I trust`）への Enter は「プロジェクトの `.claude/settings.json` の権限を確認なしで適用する」承認そのものなので、自動送信するのは `recover --auto-confirm` を明示したときだけです。
restore は trust 確認を自動承認せず、`N 個のセッションが確認ダイアログ待ちです（trust 確認は自動承認しません）` と表示して人間の応答に委ねます（`tmux attach` して内容を確認してから応答してください）。
restore の RC 登録確認ループ中も 2 秒ごとにこのダイアログ処理を行います（同じく trust は自動承認しません）。

**RC は起動時に明示します**:
restore / recover は `claude --resume <sessionId> --remote-control <会話名>` で起動します。
`remoteControlAtStartup` の設定に依存しないため、設定が無い環境でもそのまま RC 接続されます。
会話名は claude.ai/code のセッション名として渡されるので、一覧で見分けが付きます。
`CLAUDE_RC_CLAUDE_ARGS` で `--remote-control` を自分で指定した場合は、そちらが使われます（二重指定にはなりません）。

**古い会話も既定では捨てません**:
save が記録するのは「実行中かつ RC 接続中」の会話だけなので、最終チャットが古くても放置された会話とは限りません。
そのため既定では日数による skip をせず、3 日以上前の会話には次のように一言添えるだけにしています:

```
  my-project: 最終チャットは 12 日前です（そのまま復帰します）
```

古い会話を復帰対象から外したい場合は、環境変数 `CLAUDE_RC_MAX_AGE_DAYS` に 1 以上を指定してください（例: `CLAUDE_RC_MAX_AGE_DAYS=7`）。

指定した場合、会話ログの最終エントリがその日数より前の会話は restore で skip します（`0` に戻すと skip しません）。

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

reconnect の対象は、tmux セッション（既定: `rc-restore`）内で **restore が起動した pane だけ**です（pane に印を付けて識別しています）。
同じ tmux セッションに手で作ったウィンドウや、通常のターミナルで開いているセッションにはキーを送りません。
そちらは手動で `/remote-control` を実行してください。
また、対象ウィンドウの入力欄に入力中の文字が残っていると送信内容が混ざるため、入力欄は空にしておいてください。

**restore は繰り返し実行しても複製を作りません**。
既に復帰済みで tmux ウィンドウが生きている会話は skip します。
RC だけが切れている場合は `reconnect` を使ってください。

restore / recover はセッションの同時起動による credential の同時発行・同時失効を緩和するため、既定で 3 件ずつ起動し、各波ごとに RC 登録を確認してから次へ進みます。
1 つの波で登録確認がタイムアウトした場合は「◯件がRC未登録のまま。続行しますか(y/N)」と確認します（`--yes` で自動続行）。
この確認は端末（`/dev/tty`）から読むため、cron などの端末が無い環境では確認できずに中断します。自動実行では `--yes` を付けてください。
波のサイズは `--wave-size <N>` または `CLAUDE_RC_WAVE_SIZE` で変更でき、`--wave-size 0` なら従来どおり全候補を一括起動して最後にまとめて確認します。
各ウィンドウの起動間隔は引き続き既定で 3 秒空けます（`CLAUDE_RC_STAGGER_SECS` で変更、`0` で無効化）。

### リモート側に切り替え前の履歴は表示されない

**ターミナル内の会話はそのまま継続しますが、復帰後のスマホ / claude.ai/code には切り替え前のやりとりは表示されません。**
これは Claude Code 側の意図的な保護で、このツールでは変えられません。

仕組みはこうです。
Claude Code は会話ログの中に「この会話はどの RC セッション・どのアカウントに紐づいていたか」を `bridge-session` レコードとして記録しています。
アカウントを切り替えたあとに resume すると、記録された所有アカウントと現在のログインが食い違うため、**新しい RC セッションは作られるものの、過去のメッセージはサーバーへ送られません**。
Claude Code 内部ではこれを「クロスアカウント抑止（cross-account suppression）」と呼んでおり、アカウント A の時代に書かれた内容をアカウント B のサーバー側ストレージへ持ち込まないための仕組みです。
[公式ドキュメント](https://code.claude.com/docs/en/remote-control#resume-outcomes)にも次のように明記されています:

> **The record names a different account**: Claude Code starts a new session without the conversation's earlier messages and without showing a message, whether or not the recorded session still exists.

さらに、この抑止が一度起きると会話ログに永続レコード（`{"type":"history-suppression", "cause":"chokepoint_veto", ...}`）が追記され、`--fork-session` の複製先にも引き継がれます。
**つまり、一度切り替え後に開いた会話は、以後どのアカウントで開いてもリモートに履歴が出ません。**
`list` はこの状態を検出して `⚑ リモート履歴は抑止済み` と表示します。

**回避策としての会話ログの書き換えは行いません。**
技術的には該当レコードを削れば抑止を外せますが、それは「組織 A の会話内容を組織 B のサーバーへ黙って送る」という、まさにこの保護が防いでいる事象を起こします。

#### 代わりに: recap で要約を送る

抑止されるのは**過去ログの一括アップロードだけ**で、RC 確立後の新しいやりとりは通常どおり双方向に同期されます。
そこで restore / switch / recover では、RC 登録を確認できた会話に既定で要約依頼を 1 通送ります。
リモート側にも文脈を残すためです:

```bash
claude-rc-reattach restore
# RC 登録を確認できた会話にだけ、既定の要約プロンプトを送信します
# （RC 未登録の会話には送りません）
```

送信しない場合は `--no-recap` を付けます。
`reconnect` は再接続した会話だけに送るため、従来どおり `reconnect --recap` のときだけ送信します。
プロンプトは環境変数 `CLAUDE_RC_RECAP_PROMPT` で変更できます（送信後すぐ Enter を打つため、必ず 1 行で書いてください）。
`reconnect` と同じくキー送信で行うため、**対象ウィンドウの入力欄は空にしておいてください**。

既定プロンプト:

```text
アカウント切替で画面上の履歴が空になっています。これまでの会話の経緯・決定事項・いま取り組んでいる作業を10行以内で要約して表示してください。
```

### recover（save し損ねた時の復旧）

Mac のクラッシュや強制終了で `save` する前にセッションが全滅した場合、manifest がないため restore は使えません。
`recover` は manifest の代わりに**会話ログそのもの**（`~/.claude/projects/*/<sessionId>.jsonl`）を見て、「直近に更新された & いま動いていない」会話を復旧候補として拾います。

```bash
# まず候補を確認する（起動はしない）
claude-rc-reattach recover --dry-run

# 直近 12 時間まで広げる
claude-rc-reattach recover --within 12 --dry-run

# 実際に復帰させる
claude-rc-reattach recover
```

```
$ claude-rc-reattach recover --dry-run
復旧候補（直近 6 時間・実行中のものを除く）:
[/Users/me/dev/my-project]
✔ [dry-run] 2026-08-15 21:10  bb3a9965-...  my-project: 検索クエリを作って
  → claude --resume bb3a9965-... -n 'my-project: 検索クエ-restored-0815'
```

- 対象は既定で**直近 6 時間**以内に更新された会話（`--within <時間>` / `CLAUDE_RC_RECOVER_HOURS` で変更）
- restore と同じ判定で**いま動いている会話は除外**します
- サブエージェントのログ（`subagents/` 配下・`agent-*`）と、発言が 1 つも無い空のログは候補にしません
- 同じプロジェクトに複数候補がある場合は、最終更新が新しい順に表示します
- 復帰先は restore と同じ tmux セッション（既定: `rc-restore`）で、セッション名も同じ規則（`-restored-MMDD`）で付きます

| オプション | 動作 |
|---|---|
| `--within <hours>` | 対象とする時間（既定: 6） |
| `--auto-confirm` | 起動後の確認ダイアログに Enter を自動送信 |
| `--limit <N>` | 起動する復元候補の上限（既定: `CLAUDE_RC_MAX_SESSIONS` / 15。`0` で無制限） |
| `--resume-mode as-is\|summary` | 「Resume full session as-is」ダイアログの選択（既定: `as-is`） |
| `--yes` | メモリ予算の確認プロンプトを省略 |
| `--no-rename` | 復帰セッションに名前を付けない |
| `--no-recap` | RC 登録後の要約依頼を送らない |
| `--dry-run` | 候補一覧だけ表示して起動しない |

**確認ダイアログについて**:
recover は manifest 経由ではないため、起動後に trust 確認（`Do you trust the files in this folder?`）や「Resume from summary」のダイアログで止まることがあります。
起動から少し待って（既定 10 秒 / `CLAUDE_RC_CONFIRM_WAIT`）各ウィンドウを覗き、ダイアログが出ていれば件数を表示します:

```
⚠ 2 個のセッションが確認ダイアログ待ちです。--auto-confirm で自動応答できます
```

`--auto-confirm` を付けると Enter を自動送信して進めます（既定の選択肢を選ぶため、最大 3 段まで）。
**どのフォルダを信頼するかを自動承認することになるため、身に覚えのないディレクトリが候補に出ていないかを `--dry-run` で確認してから使ってください。**

ダイアログ待ちが残っていても、そこで打ち切らずに RC 登録の確認と recap まで進めます（1 件のダイアログ待ちで、既に RC 接続できている他の会話の要約まで送られなくなるのを避けるため）。
ダイアログ待ちがあった場合の終了コードは `1` です。

### prune（古い復元 pane の店じまい）

```bash
# まず候補だけ確認する
claude-rc-reattach prune --dry-run

# 6時間以上更新されていない復元 pane を閉じる
claude-rc-reattach prune

# 12時間以上に広げ、確認を省略する
claude-rc-reattach prune --idle 12 --yes
```

`prune` は tmux の復元用セッション（既定: `rc-restore` / `CLAUDE_RC_TMUX_SESSION`）配下の pane だけを見ます。
通常のターミナルで動いている Claude Code や、別の tmux セッションには触りません。

各 pane について会話IDを「いま動いている Claude Code の状態ファイル → `@rc_sid` → `--resume` 引数」の順で特定し、対応する会話ログ（`~/.claude/projects/*/<sessionId>.jsonl`）の最終更新が `--idle` 時間より古いものを列挙します。
状態ファイルを優先するのは、`--force` で `--fork-session` 復帰した pane では走っている会話IDが `@rc_sid`（復元元のID）と別物になり、`@rc_sid` の会話ログが fork 時点で止まるためです（作業中の pane を idle と誤判定して閉じないようにしています）。
実行時は `N件を店じまいします(会話の記録は残り、いつでも restore/recover で戻せます)` と確認し、対象 pane に `C-c` → `/exit` → Enter を送ります。
確認ダイアログ（trust 確認 / `Resume full session as-is` / `Enter to confirm`）が出ている pane は、キーを送らずスキップして人間に返します。
ダイアログで止まった pane は会話が進まないぶん会話ログが古いままで prune 候補の上位に来ますが、そこへ Enter を送ると restore / recover が人間の判断に委ねたはずの trust 承認を prune が通してしまうためです。
5秒待って残っている場合だけ、その pane を `kill-pane` で閉じます（window 単位で落とすと、ユーザーが `Ctrl-b "` で分割して下段で回している作業まで巻き添えで殺すため）。
分割していない window は、対象 pane を閉じた結果として window ごと消えます。

| オプション | 動作 |
|---|---|
| `--idle <hours>` | 対象にする最終更新の古さ（既定: 120=5日） |
| `--dry-run` | 候補一覧だけ表示して終了処理はしない |
| `--yes` | 店じまい確認を省略 |

### 権限モードの引き継ぎ（--dangerously-skip-permissions 等）

`--dangerously-skip-permissions` などの権限モードは、会話ではなく**プロセス起動時の状態**のため、`--resume` では引き継がれません。
そこで save の時点で元プロセスの起動引数を読み、restore で自動的に付け直します（既定で有効）。

```
$ claude-rc-reattach save
✔ 2 件の会話を保存しました → ...
  • my-project  [/path/to/project]  権限モード: --permission-mode plan

$ claude-rc-reattach restore
  my-project: 権限モードを引き継ぎます（--permission-mode plan）
```

引き継ぎ対象は `--dangerously-skip-permissions` / `--allow-dangerously-skip-permissions` / `--permission-mode` / `--permission-prompt-tool` です。
`--model` などそれ以外のオプションは引き継ぎません。
`--permission-mode` の値は既知のモード名（`default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan`）のときだけ拾います。

**bypass 系だけは既定で引き継ぎません**:
検出は `ps` の起動引数を空白で分割して行うため、`--append-system-prompt` の値に同じ語が入っていると誤検出し得ます（引用符の情報は `ps` では失われます）。
誤って別アカウントのリモート操作可能なセッションに bypass が付くのは影響が大きいため、`--dangerously-skip-permissions` / `--allow-dangerously-skip-permissions` / `--permission-mode bypassPermissions` はオプトインにしています:

```bash
claude-rc-reattach restore --inherit-bypass   # bypass も引き継ぐ
```

**有効にした方法によって挙動が変わります**:

| bypass を有効にした方法 | 復帰後 |
|---|---|
| `claude --dangerously-skip-permissions` で起動 | save 時に検出されるが、**引き継ぐには `--inherit-bypass` が必要** |
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
| `CLAUDE_RC_CLAUDE_ARGS` | （なし） | restore / recover 時に `claude` コマンドへ追加するオプション（スペース区切りで複数可。値にスペースを含む場合は `'...'` か `"..."` で囲む） |
| `CLAUDE_RC_MAX_AGE_DAYS` | `0` | 最終チャットからこの日数を超えた会話を restore で skip する（`0` = skip しない。1 以上で有効） |
| `CLAUDE_RC_VERIFY_TIMEOUT` | `60` | restore / reconnect 後に RC 登録を確認する最大秒数（`0` で確認しない） |
| `CLAUDE_RC_STAGGER_SECS` | `3` | restore で各セッションを起動する間隔の秒数（credential の同時発行・同時失効の緩和用。`0` で無効化） |
| `CLAUDE_RC_RECAP_PROMPT` | `アカウント切替で画面上の履歴が空になっています。これまでの会話の経緯・決定事項・いま取り組んでいる作業を10行以内で要約して表示してください。` | recap で送るプロンプト（必ず 1 行で書くこと） |
| `CLAUDE_RC_RECOVER_HOURS` | `6` | recover が対象とする時間（`--within` を付けた場合はそちらが優先） |
| `CLAUDE_RC_CONFIRM_WAIT` | `10` | recover で確認ダイアログを見に行くまでの待ち秒数 |
| `CLAUDE_RC_MAX_SESSIONS` | `15` | restore / recover で起動する復元候補の既定上限（`0` = 無制限） |

## 仕組み

RC セッションは**サーバー側でアカウントに紐づいている**ため、アカウントを切り替えると旧アカウントの RC セッションは掴み直せません。
一方、**会話履歴そのものはローカル**（`~/.claude/projects/`）に保存されており、アカウントとは無関係です。
このツールはその性質を利用します。

```
切り替え前                          切り替え後
─────────                          ─────────
save: RC接続中の会話を記録    →    restore: 各会話を claude --resume
（~/.claude/sessions/ を走査）      --remote-control で tmux 内に立ち上げ直す
                                    → 新アカウント配下の新しい RC セッション
                                      として再登録される
                                      （ターミナル内の会話は継続。ただしリモート側には
                                        切り替え前の履歴は表示されない）
```

RC 接続中かどうかは、Claude Code が実行中セッションごとに書き出す `~/.claude/sessions/<PID>.json` の `bridgeSessionId` フィールドの有無で判定しています。

権限モードはこの状態ファイルには含まれないため、save 時に同ファイルの `pid` から `ps` で起動引数を読み、権限関連のフラグだけを manifest に記録しています。

`recover` は manifest も状態ファイルも使わず、会話ログ（`~/.claude/projects/*/<sessionId>.jsonl`）の更新時刻を直接見ます。
状態ファイルは Claude Code の終了時に消えるため、クラッシュ後に残っているのは会話ログだけ、という前提の復旧経路です。

## 制約・知っておくべきこと

- **RC セッション ID は新規になります。**
  claude.ai/code 上では「新しいセッション」として表示されます（ターミナル内の会話は継続）。
  旧アカウント側のセッション一覧には非アクティブなエントリが残りますが、これは仕様上消せません
- **リモート側には切り替え前のやりとりが表示されません。**
  Claude Code のクロスアカウント抑止によるもので、このツールでは変えられません。
  復帰後の新しいやりとりは通常どおり同期されるため、既定の recap で要約を 1 通送って文脈を残します（[詳細](#リモート側に切り替え前の履歴は表示されない)）
- **Claude Desktop アプリのアカウントは切り替わりません。**
  Desktop は CLI とは別に認証を持つ（独自の OAuth トークンと同梱の Claude Code を使う）ため、`claude auth logout` / `login` の影響を受けません。
  Desktop も使っている場合は、アプリ側でサインアウト → 新アカウントでサインインしてください。
  Desktop の Code タブのセッション履歴も CLI とは別管理で、このツールの復帰対象にはなりません
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
- `~/.claude/sessions/<PID>.json` は非公開の内部仕様です（v2.1.233 で動作検証）。
  Claude Code のアップデートで形式が変わる可能性があります。
  動かなくなったら `claude-rc-reattach list` の結果と `ls ~/.claude/sessions/` を見比べてください

## トラブルシューティング

| 症状 | 確認すること |
|---|---|
| `list` に何も出ない | 各セッションのフッターに `/rc active` が出ているか。出ていなければそのセッションは RC 未接続（`/remote-control` で手動接続するか、`remoteControlAtStartup: true` を設定して開き直す） |
| restore 後に `/rc active` が出ない | restore の RC 登録確認で ⚠ になった会話は `claude-rc-reattach reconnect` で再接続を試す。それでもだめなら `claude doctor` で RC の適格性チェックを確認。`ANTHROPIC_API_KEY` が環境にセットされていると RC が無効になるため `unset` する |
| 復帰後しばらくして `Remote Control disconnected — Transport closed: worker credential expired or rejected (code 4094)` や `could not fetch fresh session credentials after code 401` が出る | RC の worker credential 更新に失敗して切断された状態。旧アカウントのまま開き続けたセッションでは logout によるトークン失効のため必ず発生する（そのセッションの RC は復旧不可。新アカウントで開き直す）。同一アカウントでも複数セッション並行時の OAuth refresh 競合で発生することがある Claude Code 側の既知問題（[#54443](https://github.com/anthropics/claude-code/issues/54443) / [#61912](https://github.com/anthropics/claude-code/issues/61912) / [#78309](https://github.com/anthropics/claude-code/issues/78309)）。tmux 内のセッションなら `claude-rc-reattach reconnect`、通常のターミナルなら該当セッションで `/remote-control` を実行して再接続する |
| ウィンドウがすぐ落ちる | tmux 3.2 以降なら異常終了した pane が残るので、そこにエラーが表示されています。`tmux attach -t rc-restore` で確認 |
| 「会話ログが見つかりません」で skip される | `~/.claude/projects/*/<sessionId>.jsonl` が存在するか確認（規則どおりの場所に無い場合は projects 配下を総当たりで探します） |
| `skip (already running)` で skip される | その会話はすでに開かれている（または復帰済み）ため、二重起動を防いで飛ばしています。それでも復帰させたい場合は `--force` |
| 「別プロセスで開いたままです」で skip される | `--force --no-fork` を付けた場合の動作です。`--no-fork` を外せば複製として復帰します |
| save し損ねたまま Mac がクラッシュした | `claude-rc-reattach recover --dry-run` で復旧候補を確認し、問題なければ `recover` を実行（上記「recover」参照） |
| 「最終チャットが N 日前」で skip される | `CLAUDE_RC_MAX_AGE_DAYS` に 1 以上を指定した場合の動作です。既定（`0`）では古い会話も復帰します |
| 復帰後に `--dangerously-skip-permissions` が外れている | bypass 系は既定で引き継ぎません。`--inherit-bypass` を付けるか、`CLAUDE_RC_CLAUDE_ARGS` で明示指定してください。セッション中に `/permissions` で変更した場合は検出できません（上記「権限モードの引き継ぎ」参照） |
| スマホ / claude.ai/code に切り替え前のやりとりが出ない | 仕様です（Claude Code のクロスアカウント抑止）。restore / switch / recover は既定で recap を送るため、対象ウィンドウの入力欄が空か確認してください（上記「[リモート側に切り替え前の履歴は表示されない](#リモート側に切り替え前の履歴は表示されない)」参照） |
| `list` に `⚑ リモート履歴は抑止済み` と出る | その会話は既にクロスアカウント抑止が確定しており、以後どのアカウントで開いてもリモートに履歴は出ません。recap で文脈を補ってください |
| Claude Desktop アプリが旧アカウントのまま | このツールは CLI の認証だけを切り替えます。Desktop はアプリ側でサインアウト → 新アカウントでサインインしてください |

## ライセンス

MIT
