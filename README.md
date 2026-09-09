# todo-baton

todo.md の項目を選んで、隣のターミナルの入力欄へ渡すピッカー。

todo.md をテキストエディタで書き足すと自動で読み直し、選んだ項目を直前に触っていたターミナルの入力欄へ流し込む。複数行の項目は改行のまま入る。送信キーは押さないので、実行するかどうかは人が決める。

## 使い方

```bash
todo-baton open        # 分割ペインを作ってそこで常駐させる
todo-baton             # このターミナルで起動する
todo-baton --list      # 項目を一覧表示するだけ
```

## todo.md の書式

```markdown
- [] やりたいこと
続きの行はここに書く
```

`[]` `[ ]` `[x]` のいずれでもよく、`x` のときだけ完了。チェックボックス行の直後から **空行 / 見出し / 次のチェックボックス行** のどれかが現れるまでが同じ項目になる。継続行のインデントは任意。

置き場所は次の順に決まる。

1. `--file` / `BATON_FILE`
2. cwd から親方向に `<ディレクトリ名>_todo.md` (既定は対象ディレクトリの親、`BATON_DIR` で置き場所を変えられる)
3. cwd から親方向に `todo.md` `TODO.md` `Todo.md` `.baton/todo.md`
4. `~/todo.md`

いずれも無いときは 2 の位置に作る。**プロジェクト内に todo.md を作らない**のが狙いで、`~/src/mkaigawa/chat/` で起動すると `~/src/mkaigawa/chat_todo.md` を使う。`BATON_DIR=~/todos` にすれば `~/todos/chat_todo.md` になり、プロジェクトごとに別ファイルのまま一箇所へ集められる。

既に 3 の場所に `todo.md` があればそれを使い続けるので、従来の置き方は壊れない。

## キー操作

| キー | 動作 |
|---|---|
| ↑↓ / jk | 選択 |
| ⏎ | 入力欄へ送る |
| s | 送信まで（Enter を押して送信） |
| y | クリップボードへコピー |
| x | 完了 / 未完了の切り替え |
| f | 完了項目の表示 / 非表示 |
| e | エディタで開く |
| r | 再読込 |
| q | 終了 |

## 送信先の入力欄が空になるとき

送信先が Claude Code で、入力欄が **vim モード**のときに起きる。

複数行の項目は行の区切りに Alt+Enter を送るが、これが ESC + Enter にばらけて届くと、受け取った Claude Code は ESC を Escape キーとして解釈して NORMAL モードに落ちる。そこから先に送った文字はコマンドとして食われるので、入力欄には何も入らない。一度こうなると次からは 1 行の項目でも入らない (入力欄をクリックして `i` を押せば戻る)。

改行キーを ESC を使わないものに替えると避けられる。

```bash
todo-baton --newline-key ctrl+j        # BATON_NEWLINE_KEY=ctrl+j でも同じ
```

`ctrl+j` は Claude Code の入力欄では改行になるが、シェルでは行の実行になる。送信先がシェルのペインなら既定の `alt+enter` のままにすること。

## バックエンド

どのターミナルへ渡すかはバックエンドが決める。起動時に環境から選び、`--backend` か `BATON_BACKEND` で固定できる。

| 名前 | 送信先 | 判定 |
|---|---|---|
| `cmux` | 同じワークスペースで選択中のターミナル | `CMUX_SURFACE_ID` があれば |
| `tmux` | 同じウィンドウのアクティブなペイン | `TMUX` があれば |

ピッカー本体はバックエンドの 6 つのメソッド (`refresh` / `send_text` / `submit` / `focus` / `base_dir` / `open_pane`) しか知らない。多重化ソフトごとの癖はそれぞれのクラスに閉じている。

### cmux の癖

- `cmux send -- <text>` は本文中の `\n` `\r` `\t` を改行・改行・Tab に変換するので、含んだまま送ると途中で送信される。バックスラッシュの直後で切って複数回に分けて送る (`CmuxBackend._split_escapes`)
- 改行を入れるには `cmux send-key --surface <id> alt+enter` を挟む。Claude Code は Option+Enter を改行として扱う (ESC + Enter にばらけると Escape として届く。「送信先の入力欄が空になるとき」を参照)
- ブラケットペースト注入 (`ESC[200~` … `ESC[201~`) は使えない。zsh には効くが Claude Code は ESC を Escape キーとして受け取る
- 分割ペインにコマンドを直接渡せないので、シェルが立ち上がるのを待って打ち込み、ピッカーが出るまで打ち直す

### tmux の癖

- `send-keys -l --` はバックスラッシュを解釈しないので、cmux のような分割送りは要らない
- 改行は `send-keys M-Enter`。ESC + CR という通常の Alt 表現で届く
- `split-window` にコマンドを直接渡せるので、打ち直しの再試行は要らない

## 環境変数

| 変数 | 意味 |
|---|---|
| `BATON_FILE` | todo.md のパス (1 ファイル固定) |
| `BATON_DIR` | `<ディレクトリ名>_todo.md` の置き場所 (既定: 対象ディレクトリの親) |
| `BATON_BACKEND` | `cmux` / `tmux` |
| `BATON_TARGET` | 送信先のターミナルを固定する |
| `BATON_NEWLINE_KEY` | 複数行の改行キー (`alt+enter` / `ctrl+j`) |

## 設定ファイル

`~/.config/baton/env` (`XDG_CONFIG_HOME` を尊重) に `KEY=value` を書くと、上の環境変数の既定値になる。シェルの rc を触らずに済む。

```sh
# ~/.config/baton/env
BATON_DIR=~/src/todo
BATON_NEWLINE_KEY=ctrl+j
```

`#` で始まる行は註釈。値の引用符は剥がす。`BATON_` で始まらないキーは読まない。**環境変数が設定されていればそちらが勝つ**ので、その場だけ変えたいときは `BATON_DIR=... todo-baton` でよい。

外部ライブラリは使わない (stdlib のみ)。プロジェクト内に `.env` を置く方式は、プロジェクトを汚さない方針に反するので採らない。

## インストール

`~/.local/bin/todo-baton` に配置してください。

```bash
chmod +x todo-baton
mv todo-baton ~/.local/bin/
```

## 要件

- Python 3.7+
- cmux または tmux
