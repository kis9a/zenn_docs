---
title: "Vim script で三角形に文字列を表示するための関数を書きました！"
emoji: "🔺"
type: "tech"
topics: ["vim", "poop"]
published: true
---

## スクリプト

### Vim script

~/.vimrc

```vim
" pyramid function
function! s:Pyramid(char)
  let strwid=strwidth(a:char)
  if strwid > 2
    echoerr("over string width, str width expected less than 2 int")
    return
  endif
  let height = g:pyramid_height
  let width = strwid * 2 * height + 2
  let &textwidth=width
  let start = line(".") + 1
  let end = start + height
  let str = ""

  for i in range(0, height)
    if i == 0
      let str=str . a:char
    else
      let str=str . a:char . a:char
    endif
    put =str
  endfor
  execute ':' . start . ',' . end . " center"
endfunction

" default pyramid height
let g:pyramid_height = 10

" command
command! -nargs=1 Pyramid call s:Pyramid(<f-args>)
command! -nargs=0 PyramidPoop call s:Pyramid(nr2char('0x1f4a9'))

" maps
nnoremap <silent> gp :PyramidPoop<CR>
```

```bash
# :Pyramid 💩
# :PyramidPoop

                  💩
                💩💩💩
              💩💩💩💩💩
            💩💩💩💩💩💩💩
          💩💩💩💩💩💩💩💩💩
        💩💩💩💩💩💩💩💩💩💩💩
      💩💩💩💩💩💩💩💩💩💩💩💩💩
    💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩
  💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩
💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩
```

## Shell script

~/.zshrc に書いてみるとこんな感じ

```bash
function pyramid() {
  if [[ -z "$1" ]]; then
    echo "first argument is required"
  elif [[ "${#1}" -gt 1 ]]; then
    echo "over string width, str width expected less than 2 int"
  else
    str="$1"
    columns="$(tput cols)"
    full=0
    if [[ -z $(LC_ALL=C grep -o '[[:print:][:cntrl:]]' <<<"$1") ]]; then
      full=1
    fi
    for ((i = 0; i < ${2:=20}; i = $i + 1)); do
      if [[ $full -eq 1 ]]; then
        printf "%*s\n" $(((${#str} + columns) / 2 - $i)) "$str"
      else
        printf "%*s\n" $(((${#str} + columns) / 2)) "$str"
      fi
      str=$str"$1$1"
    done
  fi
}
```

```bash
# pyramid "$(echo -e "\U1F4A9")" 10

                  💩
                💩💩💩
              💩💩💩💩💩
            💩💩💩💩💩💩💩
          💩💩💩💩💩💩💩💩💩
        💩💩💩💩💩💩💩💩💩💩💩
      💩💩💩💩💩💩💩💩💩💩💩💩💩
    💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩
  💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩
💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩
```
