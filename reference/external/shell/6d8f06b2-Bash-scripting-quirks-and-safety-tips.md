---
title: Bash scripting quirks and safety tips
source: https://jvns.ca/blog/2017/03/26/bash-quirks/
kind: external
domain: shell
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, shell]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2017/03/26/bash-quirks/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# Bash scripting quirks and safety tips

Yesterday I was talking to some friends about Bash and I realized that, even though
I’ve been using Bash for more than 10 years now there are still a few basic
quirks about it that are not totally obvious to me. So as usual I thought
I’d write a blog post.

We’ll cover

- some bash basics (“how do you write a for loop”)
- quirky things (“always quote your bash variables”)
- and bash scripting safety tips (“always use `set -u`”)

If you write shell scripts and you don’t read anything else in this post, you
should know that there is a shell script linter called
**[shellcheck](https://www.shellcheck.net/#)**. Use it to make your shell
scripts better!

We ’re going to talk about bash like it’s a programming language, because,
well, it is. The goal of this post really is not to go into details of bash
programming. I do not do complicated programming in bash and do not really plan
to learn how to. But after thinking about it a bit today, I think it’s useful
to explicitly write down some of the basics of the bash programming language.
And some things about the bash programming languages are quite different
from other programming languages I use!

I really thought I knew this stuff already but I learned a couple things by
writing this post so maybe you will too.

In bash variable assignment looks like this:

```
VARIABLE=2
```

and you reference variables with `$VARIABLE`. It’s very important that you
don’t put spaces around the = sign – `VARIABLE= 2`, `VARIABLE = 2`, and
`VARIABLE =2` are not syntax errors, but will all do different unwanted things
(like try to run a program called `2` an environment variable `VARIABLE` set to
the empty string).

Bash variables don’t need to be all-caps but they usually are.

Most bash variables you’ll use are strings. There are also some array variables
in bash but I don’t really understand those.

Sometimes I have a variable containing a string like `file.txt` and I want to
do like

```
mv $MYVAR $MYVAR__bak # wrong!
```

This code as is won’t work! It will instead look for the variable `MYVAR__bak`
which is not a real variable.

To get around this, all you need to know is that `${MYVAR}` does the same thing
as `$MYVAR`. So we can run

```
mv $MYVAR ${MYVAR}__bak # right!
```

Next, Bash has 3 kinds of variables. The kind I usually think of first (and probably use the most often) are **environment variables**.

Every process on Linux actually has environment variables (you can run `env` to
see what variables are currently set), but in Bash they’re much more easily
accessible. To see the environment variable called `MYVAR` you can run.

```
echo "$MYVAR"
```

To set an environment variable, you need to use the `export` keyword:

```
export MYVAR=2
```

When you set an environment variable, all child processes will see that
environment variable. So if you run `export MYVAR=2; python test.py`, the
python program will have MYVAR set to 2.

The next kind of variable is the **global variable**. You assign these just
like we described up above.

```
MYVAR=2
```

They behave like global variables in any other programming language.

There are also **local variables**, which are scoped to only exist inside a
bash function. I basically never use functions so (unlike in literally every
other programming language I write in) I have never used local variables.

Here’s how I write for loops in bash. This loop prints the numbers from 1 to 10.

```
for i in `seq 1 10` # you can use {1..10} instead of `seq 1 10` 
do     
 echo "$i"
done
```

If you want to write this loop on one line it looks like this:

```
for i in `seq 1 10`; do echo $i; done
```

I find this impossible to remember (how are you supposed to remember that
there’s a semicolon after `seq 1 10` but none after `do`?) so I don’t try to
remember that.

You can also write while loops but I never do that.

The cool thing about this is that you can iterate over the output of another
command. `seq 1 10` prints the numbers from 1 to 10 (one per line), and this
for loop is just taking that output and iterating over it. I use this a fair
amount.

You can interpolate command output with either backticks or `$()`.

```
OUTPUT=`command`
# or 
OUTPUT=$(command)
```

If statements in bash are pretty annoying to remember how to do. You have to
put in these square brackets, and there have to be spaces around the square
brackets otherwise it doesn’t work. `[[` and `[` square brackets
(double/single) both work. Here we get truly into bash quirk territory: `[` is
a program (`/usr/bin/[`) but `[[` is bash syntax. `[[` is better.

```
if [[ "vulture" = "panda" ]]; then
 echo expression evaluated as true
else
 echo expression evaluated as false
fi
```

Also, you can check for things like “this file exists”, “this directory exists”, etc. For example you can check whether the file /tmp/awesome.txt exists like this:

```
If [[ -e /tmp/awesome.txt ]]; then
  echo "awesome"
fi
```

This is sometimes useful but I have to look up the syntax every single time.

If you want to try out conditions from the command line you can use the `test`
command, like `test -e /tmp/awesome.txt`. It’ll return 0 for success, and an
error return code otherwise.

One last thing about why `[[` is better than `[`: if you use `[[`, then you can use `<` to do
comparisons and it won’t turn into a file redirection.

```
$ [ 3 < 4 ] && echo "true"
bash: 4: No such file or directory
$ [[ 3 < 4 ]] && echo "true"
true
```

Also one extra last thing about if: I learned today that you don’t need `[[` or
`[` to do if statements: any valid command will work. So you can do

```
if grep peanuts food-list.txt
then
echo "allergy allert!"
fi
```

Defining and calling functions in bash (especially if they have no parameters)
is surprisingly easy.

```
my_function () {
 echo "This is a function"; 
}
my_function # calls the function
```

Another bash trick: never use a variable without quoting it. Here, look at this
totally reasonable-looking shell script:

```
X="i am awesome"
Y="i are awesome"
if [ $X = $Y ]; then
 echo awesome
fi
```

If you try to run this script, you will get the incomprehensible error message
`script.sh: line 3: [: too many arguments`. What?

Bash interprets this if statement as `if [ i am awesome == i are awesome]`,
which doesn’t really make sense because there are 6 strings (i, am, awesome, i,
are, awesome). The correct way to write this is

```
X="i am awesome"
Y="i are awesome"
if [ "$X" = "$Y" ]; then # i put quotes because i know bash will betray me otherwise
 echo awesome
fi
```

There are cases where it’s okay to just use $X instead of “$X”, but can you
really keep track of when it’s okay and when it’s not okay? I sure can’t.
Always quote your bash variables and you’ll be happier.

Every Unix program has a “return code” which is an integer from 0 to 127. 0
means success, everything else means failure. This is relevant in bash because:
sometimes I run a program from the command line and want to only run a second
program if the first one succeeded.

You can do that with `&&`!

For example: `create_user && make_home_directory`. This will run `create_user`,
check the return code, and then run `make_home_directory` only if the return
code is 0.

This is different from `create_user; make_home_directory` which will run
`make_home_directory` no matter what the return code of `create_user` is

You can also do `create_user || make_home_directory` which will run
`make_home_directory` only `create_user` **fails**. That’s definitely in the
realm of clever tricks.

I’m not going to say much about job control here but: in bash you can start a background process like this

```
long_running_command &
```

If you later have regrets about backgrounding the process and want to bring it
back to the foreground, you can do that with `fg`. If there’s more than one of
those processes, you can see them all with `jobs`. For some reason `fg` takes a
“job ID” (which is what `jobs` prints) instead of a PID. Who knows. Bash.

Also, if you background LOTS of processes, the `wait` builtin will wait until they all
return.

Speaking of having regrets – if you accidentally start a process in the wrong
terminal, Nelson Elhage has a cool project called
[reptyr](https://github.com/nelhage/reptyr) that can save your process and move
it into a screen session or something.

When I write programs, usually I make mistakes. In most programming
languages I use, when something goes horribly wrong the program will exit and
tell me what went wrong.

In a bash script, you are usually running a lot of programs. Sometimes those
programs will exit with a failure return code. By default, when a program
fails, Bash will just keep running.

For example, in this script Python will fail (because the file I’m trying run
doesn’t exist) and then it’ll happily continue and print “done”.

```
python non_existant_file.py
echo "done"
```

This is almost never what I want – if one of the programs in my script fails,
I do not want it to just merrily keep going and do possibly undefined /
questionable things! That is terrifying. `set -e` will make the script stop and
hopefully prevent any damage. Here’s the safer version of that

```
set -e # put this at the beginning of your file

python non_existant_file.py
echo "done"
```

In most programming languages I use, I get an error if I try to use an unset variable. Not in Bash! By default, unset variables are just evaluated as if they were the empty string. The result of this is

```
rm -rf "$DIRECTORY/*"
```

will actually run `rm -rf /*` if `$DIRECTORY` is unset. If you use `set -u`
bash will stop and fail if you try to use an unset variable.

`set -x` will make bash print out every command it runs before running it. This
is really useful for debugging.

You can see all the other bash options you can set with `set -o`.

A lot of shell scripts I see people using in practice start with `set -eu` or
`set -eux`. Safety!

You can also `set -o pipefail` to exit if one part of a pipe fails.

Very recently I learned that there is a bash linter to help detect all these
weird quirks and more!! It ’ s called `shellcheck` and you can install it with
`apt-get install shellcheck`.

Also it has a website! <https://www.shellcheck.net/>. There is an example so can
see the kind of errors it tells you about. It’s pretty awesome and I’m excited
about trying it out.

Shellcheck knows about way way more bash scripting best practices than I do :).
When looking at the examples I was like “wow, that makes sense but I would
have never thought of that”.

There’s also a shell formatter called `shfmt` which seems useful:
<https://github.com/mvdan/sh>.

I think those are all the basic bash quirks I know! I suppose it is possible to
rant about how bash is a Terrible Programming Language that you shouldn’t be
using and why don’t we have a shell programming language that is less
confusing, but it doesn’t bother me too much.

I just try to not write very complicated bash scripts, stick to some of the
best practices here, and don’t worry too much about it. And it’s kind of
interesting to learn about the weird quirks, anyway!

If you liked this, people linked me to a bunch of other bash resources which I
will share with you now

(if we’re talking about alternative shells, though – the shell I actually use
day to day is `fish`. Fish is wonderful and I love it because of its [amazing autocomplete](https://fishshell.com/docs/current/tutorial.html#tut_autosuggestions). But I still generally write shell scripts in bash.)

thanks to Mat, Marina, Kamal, Geoffrey, Panashe, @gnomon, and Iain for talking about Bash with me!
