---
title: "MacOS の skim で PDF のリロード"
emoji: "🗒"
type: "tech"
topics: ["shell", "skim", "osascript"]
published: false
---

### MacOS の skim で PDF のリロード

[Skim / Wiki / TeX_and_PDF_Synchronization](https://sourceforge.net/p/skim-app/wiki/TeX_and_PDF_Synchronization/)

![skim gif](/images/skim.gif)

##### OSAscript

```applescript
#!/bin/bash
FOCUS=false
PDF_FILE=""

open_skim() {
	/usr/bin/osascript <<EOF
  set theFile to POSIX file "$PDF_FILE" as alias
  tell application "Skim"
    activate
    set theDocs to get documents whose path is (get POSIX path of theFile)
    if (count of theDocs) > 0 then revert theDocs
      open theFile
  end tell
  tell application "$CURRENT_TERM"
    if ${FOCUS} then
    else
      activate
    end if
  end tell
EOF
}

current_term() {
	/usr/bin/osascript <<EOF
  tell application "System Events"
    set activeApp to name of first application process whose frontmost is true
  end tell
  return {activeApp}
EOF
}
```

##### Vim

```vim
function! s:skimPDFLatex()
  let absolutePath=expand('%:p')
  let cmd = "AsyncRun skim -s pdflatex " . absolutePath
  silent execute cmd
endfunction

function! s:skimRPlot()
  let absolutePath=expand('%:p')
  let cmd = "AsyncRun skim -s rplot " . absolutePath
  silent execute cmd
endfunction

function! s:skimRMarkdown()
  let absolutePath=expand('%:p')
  let cmd = "AsyncRun skim -s rmd " . absolutePath
  silent execute cmd
endfunction

autocmd BufEnter *.rmd nnoremap <silent> sk :call <SID>skimRMarkdown()<CR>
autocmd BufEnter *.r nnoremap <silent> sk :call <SID>skimRPlot()<CR>
autocmd BufEnter *.tex nnoremap <silent> sk :call <SID>skimPDFLatex()<CR>
```

##### Source code

```bash
#!/bin/bash

# this script convert .pdf and reload skim.app.
# you can expansion, resolve path, customize converter.
# referenced: https://sourceforge.net/p/skim-app/wiki/TeX_and_PDF_Synchronization.

open_skim() {
	/usr/bin/osascript <<EOF
  set theFile to POSIX file "$PDF_FILE" as alias
  tell application "Skim"
    activate
    set theDocs to get documents whose path is (get POSIX path of theFile)
    if (count of theDocs) > 0 then revert theDocs
      open theFile
  end tell
  tell application "$CURRENT_TERM"
    if ${FOCUS} then
    else
      activate
    end if
  end tell
EOF
}

current_term() {
	/usr/bin/osascript <<EOF
  tell application "System Events"
    set activeApp to name of first application process whose frontmost is true
  end tell
  return {activeApp}
EOF
}

# Initialize global variables
SILENT=false
CURRENT_TERM="$(current_term)"
FOCUS=false
POSITIONAL_ARGS=()
FILE=""
FILE_BASE=""
FILE_NAME=""
FILE_EXTENSION=""
DIR_BASE=""
PDF_FILE=""

# parse flags
while [[ $# -gt 0 ]]; do
	case $1 in
	-s | --silent)
		SILENT=true
		shift
		;;
	-f | --focus)
		FOCUS=true
		shift
		;;
	*)
		POSITIONAL_ARGS+=("$1")
		shift
		;;
	esac
done

set -- "${POSITIONAL_ARGS[@]}"

run_skim() {
	FILE="$2"
	if [[ ! -e "$FILE" ]]; then
		printf "File not found %s" "$FILE" 1>&2
	fi

	DIR_BASE="$3"
	if [[ -z "$DIR_BASE" ]]; then
		DIR_BASE="/tmp"
	fi
	FILE="$(realpath "$FILE")"
	FILE_NAME="$(basename "${FILE%.*}")"
	FILE_EXTENSION="${FILE##*.}"

	case "$1" in
	"pdflatex")
		if [[ $(echo "$FILE_EXTENSION" | tr "[:lower:]" "[:upper:]") != "TEX" ]]; then
			printf "File extension is not .tex %s" "$FILE_BASE" 1>&2
			exit 1
		fi
		FILE_BASE="$FILE_NAME.pdf"
		PDF_FILE="$DIR_BASE/$FILE_BASE"
		pdflatex -output-directory=${DIR_BASE} -interaction nonstopmode "$FILE" && bibtex "$FILE"
		if [[ -e "$PDF_FILE" ]]; then
			open_skim
		else
			printf "File not found %s" "$PDF_FILE" 1>&2
		fi
		;;
	"rplot")
		if [[ $(echo "$FILE_EXTENSION" | tr "[:lower:]" "[:upper:]") != "R" ]]; then
			printf "File extension is not .r %s" "$FILE_BASE" 1>&2
			exit 1
		fi
		cp -f "$FILE" "$DIR_BASE"
		FILE="$FILE_NAME.r"
		FILE_BASE="$FILE_NAME.pdf"
		PDF_FILE="$DIR_BASE/Rplots.pdf"
		# shellcheck disable=SC2164
		cd "$DIR_BASE"
		rscript "$FILE"
		if [[ -e "$PDF_FILE" ]]; then
			open_skim
		else
			printf "File not found %s" "$PDF_FILE" 1>&2
		fi
		;;
	"rmd")
		if [[ $(echo "$FILE_EXTENSION" | tr "[:lower:]" "[:upper:]") != "RMD" ]]; then
			printf "File type not .rmd %s" "$FILE_BASE" 1>&2
			exit 1
		fi
		FILE_BASE="$FILE_NAME.pdf"
		PDF_FILE="$DIR_BASE/$FILE_NAME.pdf"
		local q
		q="$(
			cat <<EOF
rmarkdown::render("$FILE", rmarkdown::pdf_document(), "$DIR_BASE/$FILE_NAME")
EOF
		)"
		R -e "$q"
		if [[ -e "$PDF_FILE" ]]; then
			open_skim
		else
			printf "File not found %s" "$PDF_FILE" 1>&2
		fi
		;;
	*)
		printf "Type not found %s\n" "$1" 1>&2
		help
		exit 0
		;;
	esac
}

if [[ $SILENT == true ]]; then
	run_skim "$@" >/dev/null
else
	run_skim "$@"
fi
```
