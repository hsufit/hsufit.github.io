---
title: gdb start
tags: gdb
---

- a simple example for a simple main

## run and stop
	- load your program with gdb
```
		bash$ gdb <program to run>
```
	- run the program
```
		(gdb) run
```
	- exit from gdb
```
		(gdb) quit
```
	- relative commands
		- load program with args: bash$ `gdb --args <program to run> <arg1> <arg2> ...`
		- run program with arument: `(gdb) r <arg1> <arg2> ...`

## tracing
	- break point
		- set: `(gdb) break <function name>`
		- info: `(gdb) info breakpoints`
		- relative commands: xbreak, tbreak, display, undisplay, commands, 
					disable, enable, clear, delete
	- watch point
		- set variable: `(gdb) watch <variable name or address>` 
		- set condition: `(gdb) watch <expression>` 
		- info: `(gdb) info watchpoints
		- relative commands: awatch, rwatch`
	- flow control
		- next step: `(gdb) next`
		- continue run: `(gdb) continue`
		- run a loop: `(gdb)until`
		- return to last frame: `(gdb) return`
	- variable control
		- set, show
		- enviroment variable: `(gdb) set env`
	- frame
		- frame, up, down, backtrace
	- show code
		- show source code around this step: `(gdb) line`
		- show disassembly: `(gdb) disas`
	- show variable
		- print
		- print array: `(gdb) print <array member>@<print count>`
		- info locals
	- show info
		- stack info: stack
		- registers: registers
		- mem
		- vector registers: vector
		- float
		- symbol: symbol
		- variables: 
		- locals
		- signals:
		- program
		- sharedlibrary
		- threads
		- breakpoints
		- watchpoints
	- attach detach
	- relative commands
		- show source code by address: `(gdb) line *<address>`
- advance usage
	- run with command files: bash$ gdb -batch -x <command file>  --args <program to run> <arg1> <arg2> ...
	- start gdb and run immediately: bash$ gdb -ex=r --args <program to run> <arg1> <arg2> ...
