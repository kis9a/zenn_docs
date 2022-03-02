---
title: "zsh 自作関数、コマンド一覧を検索して補完する"
emoji: "🐚"
type: "tech"
topics: ["shell", "zsh"]
published: false
---

## イメージ

![/images/zsh-completion-function.gif](/images/zsh-completion-function.gif)

## インストール

```
brew install fzf
```

## スクリプト

~/.zshrc

```
function z_funcs() {
  if [[ -z "$1" ]]; then
    functions | grep \(\) | grep -v '^\_' | grep -v '^\s' | grep '^[a-z]' | cut -f 1 -d " " &&
      alias | cut -f 1 -d "="
  fi
  while getopts ":a" option; do
    case "$option" in
    a)
      functions | grep \(\) | grep -v '^\_' | grep -v '^\s' | grep '^[a-z]' | cut -f 1 -d " " &&
        alias | cut -f 1 -d "=" && bash -c "compgen -c"
      ;;
    *)
      cat <<'EOF'
USAGE:
z_funcs [option]
  -a: all commands
EOF
      ;;
    esac
  done
}

function z_funcf() {
  BUFFER="$BUFFER$(z_funcs | uniq | sort | fzf -m)"; zle end-of-line;
}
bindkey '^o' z_funcf
zle -N z_funcf

function z_funcf_all() {
  BUFFER="$BUFFER$(z_funcs -a | uniq | sort | fzf -m)"; zle end-of-line;
}
bindkey '^k' z_funcf_all
zle -N z_funcf_all

function zh() {
  funcs=(`z_funcs | uniq | sort | fzf -m | tr "\n" " "`)
  for f in $funcs; do
    alias "$f"
    functions "$f"
  done
}
```
