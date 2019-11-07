---
title: linux man page
tags:
---
Debian the commands would be
apt-get install man-db
apt-get install manpages-dev

## for completions
apt-get install glibc-doc


if man is not complete (such as no stdio)
just install manpages-dev

reference:
https://blog.gtwang.org/linux/linux-man-page-command-examples/
suport regular expression
man -aw <topic>
man -a <topic>
man -k <topic>


----colorful
# Less Colors for Man Pages
export LESS_TERMCAP_mb=$'\E[01;31m'       # begin blinking
export LESS_TERMCAP_md=$'\E[01;38;5;74m'  # begin bold
export LESS_TERMCAP_me=$'\E[0m'           # end mode
export LESS_TERMCAP_se=$'\E[0m'           # end standout-mode
export LESS_TERMCAP_so=$'\E[38;5;246m'    # begin standout-mode - info box
export LESS_TERMCAP_ue=$'\E[0m'           # end underline
export LESS_TERMCAP_us=$'\E[04;38;5;146m' # begin underline

sudo apt-get install most
export PAGER="most"
man printf


