---
title: "Terraform のドキュメントを早く開くためのシェルスクリプトを書きました"
emoji: "🐚"
type: "tech"
topics: ["shell", "bash", "terraform"]
published: false
---

## はじめに

Terraform のドキュメントを早く開くためのシェルスクリプトを書きました。
リポジトリ: [GitHub - kis9a/tfref: Open terraform reference easily.](https://github.com/kis9a/tfref)

最近、terraform を書くことが多いのですが、<https://registry.terraform.io/providers/hashicorp/aws/latest/docs> $provider/docs を開いて検索して...リソースのドキュメントを探して...ようやく見れる...が面倒になたので、vim から :!tfref ファイル名 ライン番号 で現在のラインより上のリソース、data 定義を元に簡単に開けるようにしてみました。 現在は、プロバイダー aws, datadog, data_datadog, data_aws しか対応してないので、GCP とか使っている方は PR いただくか ご自分で改造お願いします。また、今後の terraform document の URL の変更等あった場合、動作しない場合には　スクリプトのみを参考にお願いします。

### インストール tfref

PATH が通っているディレクトリにファイルを追加して、実行権限を付与します。

```bash
sudo curl -s https://raw.githubusercontent.com/kis9a/tfref/main/tfref > /usr/local/bin/tfref
chmod +x /usr/local/bin/tfref
```

### vim から呼び出す

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

### 使用方法

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

### 開発ツール

linter: https://github.com/koalaman/shellcheck  
formatter: https://github.com/mvdan/sh

### テスト

OSX (Darwin Kernel Version 21.2.0) と Ubuntu 20.04 (ami-0ec4d40472158dbd2) で GREEN 確認

```bash
git clone https://github.com/kis9a/tfref
cd tfref ./tfref -t
```
