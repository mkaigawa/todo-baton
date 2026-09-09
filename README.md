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
| y | クリップボードへコピー |
| x | 完了 / 未完了の切り替え |
| f | 完了項目の表示 / 非表示 |
| e | エディタで開く |
| r | 再読込 |
| q | 終了 |

## バックエンド

どのターミナルへ渡すかはバックエンドが決める。起動時に環境から選び、`--backend` か `BATON_BACKEND` で固定できる。

| 名前 | 送信先 | 判定 |
|---|---|---|
| `cmux` | 同じワークスペースで選択中のターミナル | `CMUX_SURFACE_ID` があれば |
| `tmux` | 同じウィンドウのアクティブなペイン | `TMUX` があれば |

バックエンドが持つ責務は **送信先を決める `resolve_target()` と入力欄へ置く `paste()` の 2 つだけ**で、ピッカー本体はこの 2 つしか呼ばない。`focus()` `base_dir()` `open_pane()` はできるものだけが実装する任意の口。zellij や wezterm を足すときは `Backend` を継承して外部コマンドを薄く包み、`BACKENDS` に加える。

送信先が Claude Code で入力欄が vim モードのときは入力欄に何も入らないことがある。入力欄をクリックして `i` を押すか、`todo-baton --newline-key ctrl+j` を試してください。

## 環境変数

| 変数 | 意味 |
|---|---|
| `BATON_FILE` | todo.md のパス (1 ファイル固定) |
| `BATON_DIR` | `<ディレクトリ名>_todo.md` の置き場所 (既定: 対象ディレクトリの親) |

## 設定ファイル

`~/.config/baton/env` (`XDG_CONFIG_HOME` を尊重) に `KEY=value` を書くと、上の環境変数の既定値になる。シェルの rc を触らずに済む。

```sh
# ~/.config/baton/env
BATON_DIR=~/todos
```

`#` で始まる行は註釈。値の引用符は剥がす。`BATON_` で始まらないキーは読まない。**環境変数が設定されていればそちらが勝つ**ので、その場だけ変えたいときは `BATON_DIR=... todo-baton` でよい。

外部ライブラリは使わない (stdlib のみ)。プロジェクト内に `.env` を置く方式は、プロジェクトを汚さない方針に反するので採らない。

## インストール

```bash
git clone git@github.com:mkaigawa/todo-baton.git
cd todo-baton
chmod +x todo-baton
mv todo-baton ~/.local/bin/
```

`~/.local/bin` が `PATH` に無ければ、シェルの rc ファイルに追加してください。

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 要件

- Python 3.7+
- cmux または tmux

## ライセンス

MIT License. 詳細は [LICENSE](LICENSE) を参照してください。
