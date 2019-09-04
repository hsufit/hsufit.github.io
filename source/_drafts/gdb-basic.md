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
	- set break point
```
		(gdb) break <function name>
```
	- flow control
		- next step: `(gdb) next`
		- continue run: `(gdb) continue`
	- show code
		- show source code around this step: `(gdb) line`
		- show disassembly: `(gdb) disas`
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
	- relative commands
		- show source code by address: `(gdb) line *<address>`
- advance usage
	- run with command files: bash$ gdb -batch -x <command file>  --args <program to run> <arg1> <arg2> ...
	- start gdb and run immediately: bash$ gdb -ex=r --args <program to run> <arg1> <arg2> ...
