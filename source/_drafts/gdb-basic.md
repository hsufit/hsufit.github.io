---
title: gdb start
tags: gdb
---

- a simple example for a simple main

- run and trace
	- run with args: bash$ gdb --args <program to run> <arg1> <arg2> ...
	- run with command files: bash$ gdb -batch -x <command file>  --args <program to run> <arg1> <arg2> ...
	- start gdb and run immediately: bash$ gdb -ex=r --args <program to run> <arg1> <arg2> ...
	- run program with arument: gdb$ r <arg1> <arg2> ...


	- break point
	- next step
	- show code
	- show disassembly

