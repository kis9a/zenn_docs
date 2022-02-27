---
title: "文字列を文字で囲むコマンドを作りました！(runeboxes)"
emoji: "💩"
type: "tech"
topics: ["bash"]
published: false
---

## はじめに

文字列を文字で囲むコマンドが欲しかったので作りました！
以前から terminal で生活しているので、いろいろなコマンドをインストールしたり、適当な shell script を書いて生活環境を向上させてきました。今回の runeboxes コマンドもそのライフサイクルの一環です。使用用途があるわからないですが、使えそうな箇所を参考にしてみてください。

また、bash script で書くのは 複数環境での互換性よりも手軽さを意識しているからです。OSX (Darwin Kernel Version 21.2.0) と Ubuntu 20.04 (ami-0ec4d40472158dbd2)で動作確認済み、shellcheck を参考に作成しています。

全角、半角等混ざった文字列を囲めるようになってます。以前、~/.zshrc に書き込んで使用していた [play-display-in-box-harfwidth.sh · GitHub](https://gist.github.com/kis9a/3a95e26ece257b65a87539cb97381181) では、半角のみの対応だったのでそこからの改善です、文字列の幅を取得するテクニックが bash script で見つかりそうになかったので、文字幅の取得に関しては、[GitHub - mattn/go-runewidth](https://github.com/mattn/go-runewidth) を使用させていただいています。

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
  -p: padding

EXAMPLE:
  echo apple | runeboxes @
  echo banana | runeboxes -p 3 \$
  echo -e "\U1F4A9" | runeboxes 金
  fortune | cowsay -W 60 | runeboxes -p 2 #
```

vim

```
:'<,'>!cat - | runeboxes
```

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

## もし、box_char_runewidth (文字列を囲む文字)の幅が 2の時
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

## frame における padding を除く空白の数
frame_empties="$((singles - $((box_char_runewidth * 2))))"

## 以上の初期化した変数によって以下を描画します。
@@@@@@@@@ # top border
@       @ # top empty frame
@ input @ # input lines
@       @ # bottom empty frame
@@@@@@@@@ # bottom border

## 以下の関数を使用します
## 詳細は https://github.com/kis9a/runeboxes/blob/main/runeboxes
display_string_n_times
display_empty_block
display_front_padding_block
display_back_padding_block
display_border_line
display_new_line
display_empty_frame_line
```

## bash に関して学習した事

### 開発ツール

linter, formatter に関しは以下のものを使うようにしています。shellcheck に従って書いて、formatter が自動的にフォーマットしてくれます。その他の細かなところはできるだけ考えないようにしてスクリプトを書いていきます。

linter: [GitHub - koalaman/shellcheck: ShellCheck, a static analysis tool for shell scripts](https://github.com/koalaman/shellcheck)  
formatter: [GitHub - mvdan/sh: A shell parser, formatter, and interpreter with bash support; includes shfmt](https://github.com/mvdan/sh)

Nvim + coc.nvim を使っているので以下のインストールと設定をしています。
:CocInstall coc-sh
:CocInstall coc-diagnostic
brew install shfmt
brew install shellcheck

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

他の関数を呼び出した時に意図しない値が入っていて意図しない条件分岐になるので local をつけている。細かい仕様は知らない。
[Bash Script の作法 - Qiita](https://qiita.com/autotaker1984/items/bc758fcf368c1a167353#関数内のローカル変数はlocalをつける)

#### echo -n の移植性の問題

最初は完全に echo -n で改行のない文字列を出力していましたが、移植性に問題があるようです。
[シェルスクリプトの echo の移植性の問題に本気で対応する - Qiita](https://qiita.com/ko1nksm/items/d0b066268cda42ff24eb)

#### runewidth "$(echo "\t")" が 0 になる

とりあえず、Tab は、スペース４つに変換しています。なんで 0 になるかわかったら追記します。

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
stdout="$(run_runeboxes "$box_char")"
echo "$stdout"
```

/dev/tty, exec とか使っても良さそう

```bash
exec > /tmp/file # redirect to file
echo stdout to /tmp/file
exec > /dev/tty # stdout to display
echo stdout to display
cat /tmp/file # stdout to file
```

#### cat コマンドだけでも楽しめる

cat だけでもって色々できるシェルスクリプト面白いと思いました。

```bash
## 普通のやつ
cat "a"

## read stdin
echo a | cat -

## here string
cat <<<"a"

## heredoc
cat <<'EOF'
a
EOF

## プロセス置換
cat <(echo a)

## fd とか exec
exec 3< <(echo a); cat <&3; exec 3>&-
```

### 終わりに

シェルスクリプトは他の言語より手軽に書けて手軽に実行できる点がいいですね！  
runeboxes コマンド役に立つかどうかはわかりませんが、必要であれば使ってみてください。
