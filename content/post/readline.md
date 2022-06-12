---
title: "What every user should know about Readline"
date: 2022-06-10T21:23:09Z
draft: true
tags: [ "readline", "development", "cli" ]
---

## What every user should know about Readline

Readline is an excellent library that is used by a large number of
command-line applications to provide powerful ways to input text. If
you find yourself holding down backspace or repeatedly pressing the up
arrow to find your last command, you're missing out on many of the
features of readline!

## Readline is everywhere

Most of the command lines you use probably have readline
installed. `bash` is likely the best known, but over 300 packages in
Debian 11 use it. `zsh` actually does not use GNU Readline, but it has
its own readline-like library called Zsh Line Editor. This has many of
the shortcuts, so try these out there, too. Here's a short list:

* sqlite3
* gdb
* ftp
* mysql
* psql

If you have a program that doesn't use readline, but you want it to,
try `rlwrap`
