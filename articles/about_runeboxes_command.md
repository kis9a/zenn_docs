---
title: "文字列を文字で囲むコマンドを書きました！(runeboxes)"
emoji: "🐚"
type: "tech"
topics: ["shell", "bash"]
published: true
---

## 背景

文字列を文字で囲むコマンドが欲しかったので作りました！
以前から terminal で開発生活しているので、いろいろなコマンドをインストールしたり、適当な shell script を書いて生活環境を向上させてきました。今回の runeboxes コマンドもそのライフサイクルの一環です。使用用途があるわからないですが、使えそうな箇所を参考にしてみてください。 また、shell script で書くのは 複数環境での互換性よりも手軽さを意識しているからです。OSX (Darwin Kernel Version 21.2.0) と Ubuntu 20.04 (ami-0ec4d40472158dbd2)で動作確認済み、shellcheck を参考に作成しています。

全角、半角等、複数幅が混ざった文字列を囲めるようになってます。以前、~/.zshrc に書き込んで使用していた [play-display-in-box-harfwidth.sh · GitHub](https://gist.github.com/kis9a/3a95e26ece257b65a87539cb97381181) では、半角のみの対応だったのでそこからの改善です、文字列の幅を取得するテクニックが shell script で見つかりそうになかったので、文字幅の取得に関しては、[GitHub - mattn/go-runewidth](https://github.com/mattn/go-runewidth) を使用させていただいています。
リポジトリ: [GitHub - kis9a/runeboxes: Display the piped string in the character box.](https://github.com/kis9a/runeboxes)

## イメージ

![runeboxes image](/images/runeboxes.png)

## インストール

文字幅の取得のためのコマンド、runewidth が必要なのでインストールします。

#### runewidth

```bash
# installation
git clone https://github.com/kis9a/go-runewidth
cd go-runewidth/cmd/runewidth
go install .

# usage
runewidth "💩" # 2
```

#### runeboxes

PATH が通っているディレクトリにファイルを追加して、実行権限を付与します。

```bash
sudo curl -s https://raw.githubusercontent.com/kis9a/runeboxes/main/runeboxes > /usr/local/bin/runeboxes
sudo chmod +x /usr/local/bin/runeboxes
```

#### 使用方法

runeboxes -h

```less
USAGE:
  runeboxes [options] <box_char>

OPTIONS:
  -h: help
  -t: test: runeboxes_test
  -p: padding size
  -x: horizontal padding size
  -y: vertical padding size

EXAMPLE:
  echo apple | runeboxes @
  echo banana | runeboxes -p 3 \$
  echo -e "\U1F4A9" | runeboxes -x 8 -y 2 金
  fortune | cowsay -W 60 | runeboxes -x 4 $(echo -e "\U1F4A9")
```

#### vim

```
:'<,'>!cat - | runeboxes
```

![runeboxes vim gif](/images/runeboxes-vim.gif)

## ロジック

```bash
# 入力文字列を受け取ります。
in="$(cat -)"

## 文字列の最大幅(max)を取得します。
## 入力文字列の行毎のリスト lines を初期化します。
while read -r l; do
  if [[ $(runewidth "$l") -gt $max ]]; then
    max=$(runewidth "$l")
  fi
  lines+=("$l")
done <<<"$in"


## 文字列を囲む文字(box_char_runewidth)の幅は 2 以下です。
if [[ "$box_char_runewidth" -gt 2 ]]; then
  err "String is over 2 runewidth"
  show_help
fi

## もし、文字列を囲む文字(box_char_runewidth)の幅が 2 の時
## 文字列の最大幅(max) を偶数にします。
if [[ "$box_char_runewidth" -eq 2 ]]; then
  if [[ $((max % 2)) -ne 0 ]]; then
    max=$((max + 1))
  fi
fi

## padding を含む ボックスの幅 に対しての 半角(single)での個数
singles="$((max + $((box_char_runewidth * 2)) + $((box_padding * 2))))"

## padding を含む ボックスの幅 に対しての 囲い文字幅(box_char_runewidth)での個数
lens="$((singles / box_char_runewidth))"

## frame における padding を含む空白の数
frame_empties="$((singles - $((box_char_runewidth * 2))))"

## 以上の初期化した変数によって以下を描画します。
@@@@@@@@@ # top border
@       @ # top empty frame
@ input @ # input lines
@       @ # bottom empty frame
@@@@@@@@@ # bottom border

## 以下の関数を使用します
## 詳細は https://github.com/kis9a/runeboxes/blob/main/runeboxes
repeat_string
empty_block
front_padding_block
back_padding_block
border_line
new_line
empty_frame_line
```

(追記)
vertical padding, horizontal padding も考慮するようにしため若干変わっています。

## 開発·ツール

linter, formatter に関しは以下のものを使うようにしています。shellcheck に従って書いて、formatter が自動的にフォーマットしてくれます。その他の細かなところはできるだけ考えないようにしてスクリプトを書いていきます。

linter: [GitHub - koalaman/shellcheck: ShellCheck, a static analysis tool for shell scripts](https://github.com/koalaman/shellcheck)  
formatter: [GitHub - mvdan/sh: A shell parser, formatter, and interpreter with bash support; includes shfmt](https://github.com/mvdan/sh)
brew install shfmt
brew install shellcheck

##### LSP

Nvim + coc.nvim を使っているので以下のインストールと設定をしています。
:CocInstall coc-sh
:CocInstall coc-diagnostic

```json
# coc-settings.json
"diagnostic-languageserver.filetypes": {
  "sh": "shellcheck",
  "bash": "shellcheck"
},
"diagnostic-languageserver.formatFiletypes": {
  "sh": "shfmt",
  "bash": "shfmt",
}
```

##### 末尾、行頭のインデント数確認するスクリプト

```bash
function fmt_space_check() {
  i=0
  file="$1"
  declare -a lines
  IFS=""
  while read -r l; do
    # check str front space
    if [[ "$l" =~ ^[[:blank:]]+ ]]; then
      if [[ "$((${#BASH_REMATCH[1]} % 2))" != 0 ]]; then
        echo "front: "$(($i+1))": $l"
      fi
    fi
    # check str back space
    if [[ "$l" =~ [[:space:]]*$ ]]; then
      if [[ "${#BASH_REMATCH[1]}" != 0 ]]; then
        echo "back: "$(($i+1))": $l"
      fi
    fi
    lines+=("$l")
    ((i++))
  done <"$file"
}
```

## テスト

runeboxes ファイルから runeboxes_test ファイルを読み込んでテストを開始します。

```bash
git clone https://github.com/kis9a/runeboxes
cd runeboxes
./runeboxes -t
```

run_runeboxes 関数に対して入出力の diff をテストします。

```bash
is_diff() {
  if [[ -z "$(diff <(echo "$1") <(echo "$2"))" ]]; then
    return 1
  else
    return 0
  fi
}

test_run_runeboxes() {
  if is_diff "$1" "$2"; then
    test_err "test_run_runeboxes: failed:\n$1"
    echo -e "expected:\n$2"
  else
    test_suc "test_run_runeboxes:\n$1"
  fi
}

testing_run() {

# ... 略
  ## zalgo
  box_padding_x=1
  box_padding_y=1
  box_char="66"
  box_char_runewidth="$(runewidth "$box_char")"
  test_run_runeboxes "$(echo -e "z͙͑̈͏͙ă̺ͫl̜͕̈́̓͜g̳o̳ͫ̎͆,̞ͮ͂̂͞ ̎͢heͦ ̞̺͖̓͆ĉ̗̿̉ó̇́̈m͉̗̏ͯ̊e̩̿ͯͩs̰͑" | run_runeboxes "$box_char")" "$(
    cat <<EOF
6666666666666666666666
66                  66
66 z͙͑̈͏͙ă̺ͫl̜͕̈́̓͜g̳o̳ͫ̎͆,̞ͮ͂̂͞ ̎͢heͦ ̞̺͖̓͆ĉ̗̿̉ó̇́̈m͉̗̏ͯ̊e̩̿ͯͩs̰͑  66
66                  66
6666666666666666666666
EOF
}
```

## シェルスクリプト

#### is_pipe

標準入力で文字列を受け取ります。パイプの判定は以下を参考にしました。
[シェルスクリプトでパイプを判断する - Qiita](https://qiita.com/b4b4r07/items/77c589f21a99db8bb682)

```bash
is_pipe() {
  if [ -p /dev/stdin ]; then
    return 0
  elif [ -p /dev/stdout ]; then
    return 0
  else
    return 1
  fi
}
```

#### 関数中の変数はとりあえず local をつける

他の関数を呼び出した時に意図しない値が入っていておかしくなるので local をつける。

- [Bash Script の作法 - Qiita](https://qiita.com/autotaker1984/items/bc758fcf368c1a167353#関数内のローカル変数はlocalをつける)

#### echo -n の移植性の問題

最初は echo -n で改行のない文字列を出力していましたが、移植性に問題があるようです。print を使いました。

- [シェルスクリプトの echo の移植性の問題に本気で対応する - Qiita](https://qiita.com/ko1nksm/items/d0b066268cda42ff24eb)

#### runewidth "$(echo "\t")" が 0 になる

とりあえず、Tab は、スペース４つに変換しています。なぜ 0 になるかわかったら追記します。

```bash
tab_to_space() {
  # shellcheck disable=SC2001
  echo "$1" | sed -e 's/\t/    /g'
}
```

#### 画面に出力する回数を減らす

以前は、run_runeboxes 関数中で各コンポーネント毎に文字を画面に出力していましたが遅い。

```bash
run_runeboxes "$box_char"
```

[display_count_down_number_1.md · GitHub](https://gist.github.com/kis9a/f0d35e91e1b24f5c07d12b1f27dcd1f8) の画像のような出力になります。一旦変数に格納してからその後、画面出力するのがいいです。

```bash
out="$(run_runeboxes "$box_char")"
echo "$out"
```

#### trap で CTRL+C シグナルを検知して、プログラム内で exit する

[shell の trap について覚え書き - Qiita](https://qiita.com/ine1127/items/5523b1b674492f14532a)

```bash
# trap
trap exit_quit QUIT SIGINT

exit_quit() {
  err "quit" 1>&2 # 標準出力を標準エラー出力へ
  exit 1
}

# fd (file descriptor)
# stdin 0
# stdout 1
# stderr 2
```

#### exit status code を管理する

exit status code は実行者が $? で確認できます。  
8bit の範囲である 0-255 の範囲で指定できます。  
0 で終了した場合は成功、それ以外は、エラーコードです。
ベストプラクティスとか調べてないですが、下のような感じだろうと検討。

```bash
## error code
err_code_invalid_option=3
err_code_invalid_arguments=4
err_code_invalid_box_char_width=5
err_code_invalid_input=6
err_code_not_found_runewidth=7
err_code_not_found_testfile=8

err_exit() {
  err "$1"
  if [[ -z "$2" ]]; then
    exit 1
  else
    exit "$2"
  fi
}

# run
if [[ -z "$box_char" ]]; then
  err_exit "Argument is required" "$err_code_invalid_arguments"
elif [[ "$box_char_runewidth" -gt 2 ]]; then
  err_exit "String is over 2 runewidth" "$err_code_invalid_box_char_width"
elif ! is_pipe; then
  err_exit "Pipe input is requird" "$err_code_invalid_input"
elif ! is_exists "runewidth"; then
  err_exit "Command not found: runewidth" "$err_code_not_found_runewidth"
else
  out="$(run_runeboxes "$box_char")"
  echo "$out"
fi
```

#### cat コマンドだけでも楽しめる

シェルスクリプト色々できて面白い。

```bash
cat "a" ## 普通のやつ

echo a | cat - ## read stdin

cat <<<"a" ## here string

cat <<'EOF' ## heredoc
a
EOF

cat <(echo a) ## プロセス置換

exec 3< <(echo a); cat <&3; exec 3>&- ## fd とか exec
```

### 終わりに

シェルスクリプトは他の言語より手軽に書けて手軽に実行できる点がいいですね  
最近、以前よりは、まともに書けるようになって来た気がします。  
runeboxes コマンドに関しては役に立つかどうかはわかりませんが、ぜひ使ってみてください！
その他には、以下のようなシェルスクリプト関連のトピックに関連するを投稿しています。  
興味があればぜひコメント等をいただけると嬉しいです。

- [zsh 関数、コマンド一覧から検索して補完する関数を書きました！](https://zenn.dev/kis9a/articles/my_zsh_completion_function)
- [Terraform のドキュメントを簡単に開くためのコマンドを書きました！(tfref)](https://zenn.dev/kis9a/articles/about_tfref_command)
- [Vim script 三角形に文字列を入力するためだけの関数を書きました！](https://zenn.dev/kis9a/articles/display_pyramid_poop_vim_script)
