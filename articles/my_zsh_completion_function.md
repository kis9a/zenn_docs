---
title: "zsh 関数、コマンド一覧を検索して補完する。"
emoji: "🐚"
type: "tech"
topics: ["shell", "zsh"]
published: true
---

## はじめに

設定した alias, 自作関数 覚えてますか？  
自分は、設定したこと自体覚えてないものがほとんどです。  
zenn の投稿者の記事や、Qiita, Stack Overflow などから便利な関数をコピペしたり、  
AWS, Datadog, Git のオペレーションのコマンドメモとかも適当に書き殴っていたら  
おかげさまで、 ~/.zshrc が 5000 行を超えてました 💦  
設定した alias を覚えるための alias が必要になりそうです。  
[dotfiles/.zshrc at kis9a · kis9a/dotfiles · GitHub](https://github.com/kis9a/dotfiles/blob/kis9a/zsh/.zshrc)

今回は、自作関数, alias, コマンド一覧などからを検索して入力を補完できるように、  
簡単ですが、個人的に使っている関数を紹介します。お役にたてれば幸いです。

## イメージ

![/images/zsh-comp-demo.gif](/images/zsh-comp-demo.gif)

## スクリプト

fzf を使用するのでインストールしましょう。

```bash
# brew の場合
brew install fzf
```

~/.zshrc に書き込んでください。

```bash
function zsh_functions() {
  if [[ -z "$1" ]]; then
    functions | grep \(\) | grep -v '^\_' | grep -v '^\s' | grep '^[a-z]' | cut -f 1 -d " " &&
      alias | cut -f 1 -d "="
  fi
  while getopts ":a" option; do
    case "$option" in
    a)
      functions | grep \(\) | grep -v '^\_' | grep -v '^\s' | grep '^[a-z]' | cut -f 1 -d " " &&
        alias | cut -f 1 -d "=" && bash -c "compgen -c"
      break
      ;;
    *)
      cat <<'EOF'
USAGE:
zsh_functions [option]
  -a: all commands
EOF
      break
      ;;
    esac
  done
}

function zh() {
  funcs=($(zsh_functions | uniq | sort | fzf -m | tr "\n" " "))
  for f in $funcs; do
    alias "$f"
    functions "$f"
  done
}

function _zsh_function_find() {
  BUFFER="$BUFFER$(zsh_functions | uniq | sort | fzf -m)"
  zle end-of-line
}
bindkey '^o' _zsh_function_find
zle -N _zsh_function_find

function _zsh_command_find() {
  BUFFER="$BUFFER$(zsh_functions -a | uniq | sort | fzf -m)"
  zle end-of-line
}
bindkey '^k' _zsh_command_find
zle -N _zsh_command_find
```

## 説明

#### zsh_functions()

zsh の関数、alias 一覧を取得します。  
-a option をつけた場合、コマンド一覧を含めて取得します。  
grep -v で内部関数や [a-z] 以外から始まるような関数は除外してます。  
bash -c "compgen -c" という、bash built-in command を使用して、コマンド一覧を取得します。

#### zh()

zsh の関数、alias 一覧をから、fzf で検索して、関数、alias の内容を表示します。  
zsh help の意味合いですが、簡単に入力しやすいように zh とだけしています。

#### \_zsh_function_find()

zsh の関数、alias 一覧をから、fzf で検索して、end-of-line に挿入します。

#### \_zsh_function_find()

zsh の関数、alias、コマンド 一覧をから、fzf で検索して、end-of-line に挿入します。

#### bindkey, zle

bindkey '^o'
bindkey '^k'

に関しては、自分が使いやすい key に設定しましょう。  
$ bindkey コマンドで設定されている bindkey 一覧が取得できます。  
普段使わないキーを潰して上の物を使用するといいでしょう。

ZLE に関しては以下の記事がわかりやすいので参考にしましょう。非常に便利です。
[コマンドライン編集機能 Zsh Line Editor を使いこなす - Qiita](https://qiita.com/b4b4r07/items/8db0257d2e6f6b19ecb9)

個人的に他の部分では以下用途などで使用しています。(zle accept-line)

https://zenn.dev/kis9a/scraps/02f3ec438d93d1

```bash
function up() {
  cd ..
  zle accept-line
}
zle -N up
bindkey '^k' up

function zf() {
  local dir=$(z | sort -rn | cut -c 12- | fzf --height 40%)
  if [ -n "$dir" ]; then
    cd $dir
    zle accept-line
  else
    return 1
  fi
}
zle -N zf
bindkey '^j' zf
```

## おわりに

この関数があれば臆せず、どんどん alias を増やせます。[とりあえず alias 書いとく。](https://gist.github.com/kis9a/939108fdb6de2eb02d4436252e552090)  
CLI が一番簡略的で強力なインターフェースだと個人的に思っているので、  
アクセスを簡単にして、生産性、実現可能性を上げていきましょう！(自分もへ)
