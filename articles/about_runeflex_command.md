---
title: "複数ファイルの文字列を並べるコマンドを書きました！(runeflex)"
emoji: "🐚"
type: "tech"
topics: ["shell", "bash"]
published: false
---

## 背景

> 文字列を文字で囲むコマンドが欲しかったので作りました！
> 以前から terminal で開発生活しているので、いろいろなコマンドをインストールしたり、適当な shell script を書いて生活環境を向上させてきました。今回の runeflex コマンドもそのライフサイクルの一環です。使用用途があるわからないですが、使えそうな箇所を参考にしてみてください。 また、shell script で書くのは 複数環境での互換性よりも手軽さを意識しているからです。OSX (Darwin Kernel Version 21.2.0) と Ubuntu 20.04 (ami-0ec4d40472158dbd2)で動作確認済み、shellcheck を参考に作成しています。

https://zenn.dev/kis9a/articles/about_runeflex_command

これの続きです。文字列を文字で囲めるようになった訳ですが、並べるのはどうすればいいんだと思って作りました。おそらく実用性は、皆無で、文字通りクソを標準出力して楽しむためのコマンドです。ほとんど以前のものと被る項目も多いため以前の記事と合わせてみていただけるといいとお思います。

> リポジトリ: [GitHub - kis9a/runeflex: Display the piped string in the character box.](https://github.com/kis9a/runeflex)

## イメージ

![runeflex image](/images/runeflex.png)

## インストール

#### runewidth

以前と同じで、文字幅の取得のためのコマンド、runewidth が必要なのでインストールします。

```bash
# installation
git clone https://github.com/kis9a/go-runewidth
cd go-runewidth/cmd/runewidth
go install .

# usage
runewidth "💩" # 2
```

#### runeflex

PATH が通っているディレクトリにファイルを追加して、実行権限を付与します。

```bash
sudo curl -s https://raw.githubusercontent.com/kis9a/runeflex/main/runeflex > /usr/local/bin/runeflex
sudo chmod +x /usr/local/bin/runeflex
```

runeflex -h

```less
USAGE:
  runeflex [options] [argument] [file1] [file...]

OPTIONS:
  -h, --help: show help
  -t, --test: test: runeflex_test
  -b, --border: border char of flexbox
  -g, --gap: gap space of flexbox
  -m, --margin: margin size
  -mx, --margin-x: horizontal margin size
  -my, --margin-y: vertical margin size
  -p, --padding: padding size
  -px, --padding-x: horizontal padding size
  -py, --padding-y: vertical padding size

EXAMPLE:
  runeflex -b # -px 2 -py 1 <(echo "1") <(echo "2\n4") <(echo "3\n5\n6")
  runeflex -g 8 -mx 4 -my 2 <(echo "1") <(echo "2\n4") <(echo "3\n5\n6")
  runeflex -b # -m 2 -py 0 <(echo "a") <(echo "bc\nd")
  runeflex -b $(echo -e "\U1F4A9") -m 0 -p 0 <(echo "a") <(echo "bc金\nd")
  runeflex -g 4 ./file1 ./file2
```

## ロジック

ロジックに関しては今回省略します。  
ただ以前のようにゴニョゴニョして出力しているだけです。  
<https://zenn.dev/kis9a/articles/about_runeboxes_command#%E3%83%AD%E3%82%B8%E3%83%83%E3%82%AF>

もっといい感じにできるとは思いますが、時間をかけない、  
最低限テストが通っているので OK という自分ルールなので数時間で書けて気が楽です。

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

## テスト

runeboxes ファイルから runeboxes_test ファイルを読み込んでテストを開始します。

```bash
git clone https://github.com/kis9a/runeflex
cd runeflex
./runeflex -t
```

run_runeflex 関数に対して入力を与えて期待するも出力と実際の出力 diff をテストします。

## 改行区切りで文字列を並べる

上のようなコマンドをつくていましたが、Vim からどう呼び出せばいいんだと思いました。  
runeflex を使うとすると複数の選択範囲をファイルとして入力しないといけません。  
ここは妥協して、runeflex を wrap して runeflexrn スクリプトを記述し、改行区切りで文字列を並べるようにしました。まず、runeflex コマンドを使用せずにプロトタイプを書いてみました。

```bash
#!/bin/bash

# flexrn

# input
## s1
## s2
##
## s3

# output
## s1 s3
## s2

IFS=""
in=$(cat -)
files=()
file_count=0
line_count=0

if [[ -z "$1" ]]; then
  CHAR=" "
else
  CHAR="$1"
fi

while read -r line; do
  if [[ "$line" =~ ^$ ]]; then
    ((file_count++))
    line_count=0
  else
    if [[ $line_count -eq 0 ]]; then
      files["$file_count"]="${files["$file_count"]}""$(printf "%s" "$line")"
    else
      files["$file_count"]="${files["$file_count"]}""$(printf "\n%s" "$line")"
    fi
    ((line_count++))
  fi
done <<<"$in"

nfiles=()
count=0
line=""
for f in "${files[@]}"; do
  line_count=0
  while read -r line; do
    if [[ $count -eq 0 ]]; then
      nfiles["$line_count"]="${nfiles["$line_count"]}""$(printf "%s" "$line")"
    else
      nfiles["$line_count"]="${nfiles["$line_count"]}""$(printf "\n%s" "$line")"
    fi
    ((line_count++))
  done <<<"$f"
  ((count++))
done

for fn in "${nfiles[@]}"; do
  line_count=0
  while read -r line; do
    if [[ "$line_count" -eq "$(wc -l <<<"$fn")" ]]; then
      printf "%s" "$line"
    else
      if [[ "$line_count" -eq 0 ]]; then
        printf "%s$CHAR" "$line"
      else
        printf "%s" "$line"
      fi
    fi
    ((line_count++))
  done <<<"$fn"
  printf "\n"
done
```

```
echo "1\n2\n\n\n3" | flexrn

# 出力
1 3
2
```

## runeflexrn

### イメージ

![runeflexrn image](/images/runeflexrn.gif)

Install runeflexrn

```
install_path=/usr/local/bin/runeflexrn
sudo curl -s https://raw.githubusercontent.com/kis9a/runeflex/main/runeflexrn > "$install_path"
chmod +x "$install_path"
```

Usage example

```
:'<,'>!runeflexrn -mx 4 -py 2 -b ab
```

## シェルスクリプト

以前のシェルスクリプト (<https://github.com/kis9a/runeboxes/blob/4048483228cb6953dcdc814f0a9ee9459c54a529/runeboxes>) よりちょっと進化した感があるので、その部分を書き足します。

### グローバルな変数は UPPER_CASE で記述したほうが綺麗

syntax hilight が聞いて色が変わったり、可読性が増しました。

before
![runeflex before lowercase](/images/runeflex_before_lower.png)

after
![runeflex after lowercase](/images/runeflex_after_upper.png)

### getopts での flag のパースをやめる

getopts の方法がベストだと思っていましたが、  
以前のよりオプションを増やしていくと流石に大変そうだったので、調べました。

以前のもの

```bash
while getopts ":htpxy" o; do
  case "${o}" in
  x)
    if [[ "$1" == "-x" ]]; then
      if [[ "$#" -lt 2 || "$#" -gt 5 ]]; then
        err_exit "Invalid -x option usage" "$err_code_invalid_option"
      fi
      if is_number "$2"; then
        box_padding_x="$2"
      else
        err_exit "Invalid -x option usage" "$err_code_invalid_option"
      fi
      if [[ -z "$3" ]]; then
        box_char="#"
      elif [[ "$3" == "-y" ]]; then
        if [[ -z "$4" || -n "$6" ]]; then
          err_exit "Invalid -x -y option usage" "$err_code_invalid_option"
        else
          box_padding_y="$4"
          if [[ -n "$5" ]]; then
            box_char="$5"
          else
            box_char="#"
          fi
        fi
      else
        if [[ -n "$4" ]]; then
          err_exit "Invalid -x -y option usage" "$err_code_invalid_option"
        else
          box_char="$3"
        fi
      fi
    fi
    break
    ;;
  y)
    # 省略
    # ...
    ;;
  *)
    err_exit "Invalid option" "$err_code_invalid_option"
    ;;
  esac
done
```

この回答のものが便利で簡単ですのでおすすめです。
<https://stackoverflow.com/questions/192249/how-do-i-parse-command-line-arguments-in-bash>

```bash
POSITIONAL_ARGS=()

while [[ $# -gt 0 ]]; do
  case $1 in
    -e|--extension)
      EXTENSION="$2"
      shift # past argument
      shift # past value
      ;;
    -s|--searchpath)
      SEARCHPATH="$2"
      shift # past argument
      shift # past value
      ;;
    --default)
      DEFAULT=YES
      shift # past argument
      ;;
    -*|--*)
      echo "Unknown option $1"
      exit 1
      ;;
    *)
      POSITIONAL_ARGS+=("$1") # save positional arg
      shift # past argument
      ;;
  esac
done

set -- "${POSITIONAL_ARGS[@]}" # restore positional parameters
```

### 終わりに

シェルスクリプトは他の言語より手軽に書けて手軽に実行できる点がいいですね  
最近、以前よりは、まともに書けるようになって来た気がします。  
runeflex コマンドに関しては役に立つかどうかはわかりませんが、ぜひ使ってみてください！
その他には、以下のようなシェルスクリプト関連のトピックに関連するを投稿しています。  
興味があればぜひコメント等をいただけると嬉しいです。

- [文字列を文字で囲むコマンドを書きました！(runeboxes)](https://zenn.dev/kis9a/articles/about_runeboxes_command)
- [zsh 関数、コマンド一覧から検索して補完する関数を書きました！](https://zenn.dev/kis9a/articles/my_zsh_completion_function)
- [Terraform のドキュメントを簡単に開くためのコマンドを書きました！(tfref)](https://zenn.dev/kis9a/articles/about_tfref_command)
- [Vim script 三角形に文字列を入力するためだけの関数を書きました！](https://zenn.dev/kis9a/articles/display_pyramid_poop_vim_script)
