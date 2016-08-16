---
title: Globs In Shell
date: 2016-08-16 16:00
---

## General

"Glob" is the common name for a set of Bash features that match or expand specific types of patterns.It is necessary to know about them. Sometimes it may helps you.


## extglob

In bash, we need to run this command first to use the extend glob

```bash
shopt -s extglob
```

And then, we can use those syntax of glob:

+ ?(pattern-list)  
  zero or one occurrence of the given patterns   
+ *(pattern-list)   
  zero or more occurrence of the given patterns   
+ +(pattern-list)   
  one or more occurrence of the given patterns    
+ @(pattern-list)   
  one of the given pattern   
+ !(pattern-list)  
  match anything except one of the given patterns   

For instance:
```bash
rm !(*.jpg) 
cp !(04 *).mp3   /mnt   # cp all song to mnt/ except one 
	 
```

There are more complicated examples.

```bash
x=${x##+([[:space:]])}; x=${x%%+([[:space:]])}
#nest of the extglob
[[ $fruit = @(ba*(na)|a+(p)le) ]] && echo "Nice fruit"
```

**extglob changes the way certain characters are parsed,It is necessary to have
a newline (not just a semicolon) between shopt -s extglob and any subsequent
commands to use it.**

You cannot enable extends globs inside a group command.  Note that the typical
function body is a group command. An unpleasant workaround could be to use
a subshell command list as the function body

## nullglob

nullglob expands non-matching globs to zero arguments, rather than to themselves.To undestand it, there is an example:

```bash
ls *.c   # if this encounter an error, mean: *.c No such file or directory
#with nullglob set
shopt -s nullglob
ls *.c  # this command will like the ls without arguments, and list everything
```

>**Warning**
>There are some bugs when we use the nullglob. all of those are about the array

To removing array elements, if you use the nullglob, and you use the `unset array[1]` that may useless. you
need to use `unset -v "array[1]"` this can work.

`nullglob` is not the specified by POSIX, so when you want to port that, you
should explicitly check that glob match.


## dotglob

With `dotglob` it will show the dot file. Note that, when dotglob is enable,`*`
will match files like `.bashrc` but not the `.` or `..` dirctories. 


## globstar

globstar recursively repeats a pattern contain "**"
Here are some examples:

```bash
$ shopt -s globstat ; tree  # this will show all level
```

if you use the `files=(**)`  this is equivalent to: files=(* */* */*/*) find all files recursively.
Just like `'*'`, `"**"` followed by a `/` will only match directories:

```bash
files=(**/)  # find all subdirectories
files=(. **/) # find all subdirectories, including the current directory
```

## failglob

If a pattern fails to match, bash report an expansion error. This can be useful at then commandline:

```bash
$ > *.foo # if glob fails to match ,create file *.foo
$ shopt -s failglob
$ > *.foo  # if can match, doesn't get executed

```

## GLOBIGNORE

This allows you to specify patterns a glob _should not match_. This lets you work around the infamous "I want to match all of my dot files, but . or .."

```bash
echo .*  # will show the . and ..
GLOBIGNORE=.:..
echo .*  # will not show . and ..
```

## nocasematch

Globs inside [[ and case commands are matched case-insensitive. means the capital and lower case alpha will work.

```bash
[[ $f= *.@(txt|jpg) ]]  # that means f can match the `txt/TXT/Txt/ etc.` so do the jpg. 
```


(Done)
