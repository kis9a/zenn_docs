---
title: "tfref Terraform のドキュメントを簡単に開くためのコマンドを書いた"
emoji: "🐚"
type: "tech"
topics: ["shell", "bash", "terraform"]
published: true
---

## はじめに

Terraform のドキュメントを簡単に開くためのシェルスクリプトを書きました。
リポジトリ: [GitHub - kis9a/tfref: Open terraform reference easily.](https://github.com/kis9a/tfref)

最近、terraform を書くことが多いのですが、<https://registry.terraform.io/providers/hashicorp/aws/latest/docs> $プロバイダーの docs を検索して開いて...リソースのドキュメントを検索して...ようやく見れる...が面倒になたので、Vim から (:!tfref ファイル名 ライン番号)で現在のラインよりも上のラインにあるリソース、data 定義を元に URL に変換して簡単に開けるようにしてみました。 現在は、プロバイダー aws, datadog, data_datadog, data_aws しか対応してないので、GCP など他のプロバイダーを使っている方は PR 頂くか、ご自分で改造お願いします。また、今後の terraform docs 自体の ページ URL 構造の変更等あった場合、動作しない場合には スクリプトのみを参考にお願いします。

追記)  
以下のプロバイダーに対応しました。

```
SUPPORTED TYPE:
aws, data_aws
google, data_google
azurerm, data_azurerm
kubernetes, data_kubernetes
datadog, data_datadog
onepassword, data_onepassword
github, data_github
```

## インストール tfref

PATH が通っているディレクトリにファイルを追加して、実行権限を付与します。

```bash
sudo curl -s https://raw.githubusercontent.com/kis9a/tfref/main/tfref > /usr/local/bin/tfref
chmod +x /usr/local/bin/tfref
```

## Vim から呼び出す

Vim ではなくても、エディターからコマンド呼び出せる方は参考に。

```vim
function! s:tfref()
  let fullpath=expand('%:p')
  let cursorline=line('.')
  let cmd = "!tfref -f \"" . fullpath . "\" " . cursorline
  silent execute cmd
endfunction

nnoremap <silent> <Leader>tf :call <SID>tfref()<CR>
```

## 使用方法

tfref -h

```less
USAGE:
  tfref [options] <resource|data>

OPTIONS:
  -h: help
  -t: test: bash tfref_test
  -f: open_read_file_line_up: tfref -f $file_path $line

SUPPORTED TYPE:
  aws, datadog, data_datadog, data_aws

EXAMPLE:
  tfref aws_instance
  tfref datadog_monitor
  tfref data.aws_instance
  tfref data.datadog_api_key
  tfref -f "./ec2.tf" 20
```

## 開発ツール

詳しい設定方法については、コメントもらえればお答えします。
**linter**: https://github.com/koalaman/shellcheck
**formatter**: https://github.com/mvdan/sh

## テスト実行

```bash
git clone https://github.com/kis9a/tfref
cd tfref ./tfref -t
```

OSX (Darwin Kernel Version 21.2.0) と Ubuntu 20.04 (ami-0ec4d40472158dbd2) で GREEN 確認

```
[02-20.22][INFO]: testing
✓  test_get_open_url: aws_instance
✓  test_get_open_url: aws_instance.some
✓  test_get_open_url: data.aws_instance
✓  test_get_open_url: datadog_monitor
✓  test_get_open_url: data.datadog_api_key
✓  test_read_line_up: tfref_test_file.tf:0
✓  test_read_line_up: tfref_test_file.tf:9
✓  test_read_line_up: tfref_test_file.tf:15
✓  test_read_line_up: tfref_test_file.tf:17
✓  test_read_line_up: tfref_test_file.tf:22
✓  test_get_open_url_status_code: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance:200
✓  test_get_open_url_status_code: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/instance:200
✓  test_get_open_url_status_code: https://registry.terraform.io/providers/datadog/datadog/latest/docs/resources/monitor:200
✓  test_get_open_url_status_code: https://registry.terraform.io/providers/datadog/datadog/latest/docs/data-sources/api_key:200
```

## スクリプト

今後変更する可能性はありますが、記事公開時点で動作確認済のものです。

#### 対応 Provider の追加で貢献する

以下のスクリプトの get_open_url(), get_data_type(), get_type() にパターンを追加後、  
tfref_test にテストケースを追加して動作確認後 PR いただけると助かります。

```bash
#!/bin/bash

show_help() {
  cat <<'EOF'

USAGE:
  tfref [options] <resource|data>

OPTIONS:
  -h: help
  -t: test: bash tfref_test
  -f: open_read_file_line_up: tfref -f $file_path $line

SUPPORTED TYPE:
  aws, datadog, data_datadog, data_aws

EXAMPLE:
  tfref aws_instance
  tfref datadog_monitor
  tfref data.aws_instance
  tfref data.datadog_api_key
  tfref -f "./ec2.tf" 20

EOF
  exit 0
}

fix_open_arg() {
  if [[ "$2" =~ ^data_.* ]]; then
    echo "$1" | cut -f 1,2 -d '.'
  else
    echo "$1" | cut -f 1 -d '.'
  fi
}

get_open_cmd() {
  case "$(uname | tr "[:lower:]" "[:upper:]")" in
  *'linux'*) echo 'xdg-open' ;;
  *'bsd'*) echo 'xdg-open' ;;
  *'darwin'*) echo 'open' ;;
  *) echo 'open' ;;
  esac
}

is_exists() {
  which "$1" >/dev/null 2>&1
  return $?
}

open_run() {
  local open_cmd
  open_cmd="$(get_open_cmd)"
  if ! is_exists "$open_cmd"; then
    err "$open_cmd command is not found"
  fi
  url="$(get_open_url "$1")"
  if [[ -n "$url" ]]; then
    $open_cmd "$url"
  fi
  exit 0
}

get_open_url() {
  local type arg
  type="$(get_type "$1")"
  arg="$(fix_open_arg "$1" "$type")"
  case $type in
  "aws")
    echo "https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/${arg//aws_/}"
    ;;
  "datadog")
    echo "https://registry.terraform.io/providers/datadog/datadog/latest/docs/resources/${arg//datadog_/}"
    ;;
  "data_aws")
    echo "https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/${arg//data.aws_/}"
    ;;
  "data_datadog")
    echo "https://registry.terraform.io/providers/datadog/datadog/latest/docs/data-sources/${arg//data.datadog_/}"
    ;;
  "*")
    err "not supported type"
    ;;
  esac
}

# Support types:
# data hoge {}
# resource fuga {}
open_read_file_line_up() {
  arg="$(read_file_line_up "$2" "$3")"
  if [[ -n "$arg" ]]; then
    open_run "$arg"
  fi
  exit 0
}

read_file_line_up() {
  local file line
  declare -a lines
  file="$1"
  line="$2"
  i=0
  while read -r l; do
    lines+=("$l")
    ((i++))
  done <"$file"

  for ((j = line; 0 < j; j--)); do
    line_type="$(get_line_type "${lines[$j - 1]}")"
    if [[ "$line_type" != "not-specified" ]]; then
      arg="$(echo "${lines[$j - 1]}" | cut -f 2 -d " " | sed -e "s/\"//g")"
      if [[ "$line_type" == "data" ]]; then
        arg="data.$arg"
      fi
      cat <<<"$arg"
      break
    fi
  done
}

get_line_type() {
  if [[ "$1" =~ ^resource.* ]]; then
    echo "resource"
  elif [[ "$1" =~ ^data.* ]]; then
    echo "data"
  else
    echo "not-specified"
  fi
}

get_data_type() {
  if [[ "$1" =~ ^data\.aws_.* ]]; then
    echo "data_aws"
  elif [[ "$1" =~ ^data\.datadog_.* ]]; then
    echo "data_datadog"
  else
    err "not supported: $1"
  fi
}

get_type() {
  if [[ "$1" =~ ^data\..* ]]; then
    get_data_type "$1"
  elif [[ "$1" =~ ^aws_.* ]]; then
    echo "aws"
  elif [[ "$1" =~ ^datadog_.* ]]; then
    echo "datadog"
  else
    err "not supported type"
  fi
}

# template
# --- template --- {{{
readonly cf="\\033[0m"
readonly red="\\033[0;31m"
readonly green="\\033[0;32m"
readonly yellow="\\033[0;33m"

err() {
  local _date
  _date=$(showdate)
  echo -e "[$_date][${red}ERROR${cf}]: $1" 1>&2
  exit 1
}

warn() {
  local _date
  _date=$(showdate)
  echo -e "[$_date][${yellow}WARNING${cf}]: $1"
}

info() {
  local _date
  _date=$(showdate)
  echo -e "[$_date][INFO]: $1 "
}

succ() {
  local _date
  _date=$(showdate)
  echo -e "[$_date][${green}SUCCESS${cf}]: $1"
}

showdate() {
  local _date
  _date=$(date +%d-%H.%M)
  echo "$_date"
}

cleanup() {
  info "cleanup.."
}
#}}}

# run
while getopts ":hft" o; do
  case "${o}" in
  h)
    show_help
    exit 0
    ;;
  t)
    if [ -f "tfref_test" ]; then
      info "testing"
      # shellcheck disable=SC1091
      . "tfref_test"
    else
      err "tfref_test is not found"
    fi
    exit 0
    ;;
  f)
    if [[ "$#" -eq 3 && "$1" == "-f" ]]; then
      open_read_file_line_up "$1" "$2" "$3"
    else
      show_help
    fi
    exit 0
    ;;
  *)
    err "invalid option"
    show_help
    exit 1
    ;;
  esac
done

if [[ "$#" -ne 1 ]]; then
  show_help
else
  open_run "$1"
fi
```
