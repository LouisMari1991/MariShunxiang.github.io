---
title: Globbing
date: 2018-06-10 20:52:19
tags: The Linux Filesystem
categories: Linux
---

## Globbing: Matching Files

Any time you want to operate on a bunch of file that have similar names, you an use glob pattern to do it.

I am not making up ths name. Globbing is the real,actual,technical term for matching files by name in the Unix shell. Seriously, globbing. If you don't believe me you can look it up, `man glob`.

Globbing is kind of pattern matching for file names. When you write a glob pattern in a shell command,the shell turns. that pattern into a list of file names that exist to match the pattern.

For instance,a star matches any string of characters. You an use a star at the beginning or at the end of a pattern.
```shell
$ ls *html
$ ls app*
```
Patterns can be all sorts of different lengths. You an use two stars in the same pattern. For instance,here,matching every file whose name contains pp.
```shell
$ ls *pp*
```
A star can appear in the middle of a pattern. Matching all the files that start with B and end with png.
```shell
$ ls b*png
```
There are other patterns you an use as well.
Foe instance,to match files thar end in either CSS or HTML,a list of strings in curly braces will match any of the alternatives.
```shell
$ ls app.{css.html}
```
A single queston mark matches any one character. So BEA question mark dot png matches both bean and bear,but it doesn't match beer or bess.
```shell
$ ls bea?.png
```
Two question marks matches two characters,and so on.
```shell
$ ls bc??.png
```
List of characters inside square brackets matches an one of the characters inside those brackets,so be and then square brackets A-E-I-O-U R dot png will match bear and beer, but not bean or bes.
```shell
$ ls be[aeiou]r.png
```
Something to watch out for is that file name in Linux are always case sensitive, and that applies to globbing too.
For instace,if you have files that end in jpg in capitals and files that end in jpg in lower case,you have to specify which one you want.
```shell
$ ls *JPG
$ ls *jpg
```
If you want both,you could use the curly braces.
```shell
$ ls *{JPG,jpg}
```

## Quiz

1. Copy all the files in the "www" directory that end in "html" to the "backup" directory.
```shell
cp www/*html backup
```
2. List all the files that end in "jpg" or "png" in the current directory.
``` shell
$ ls *{jpg,png}
$ ls *jpg *png
$ ls *{jp,pn}g
```
3. Print "Short names:" followed by all the one-character filenames int current directory.
```
$ echo Short names: ?
```


