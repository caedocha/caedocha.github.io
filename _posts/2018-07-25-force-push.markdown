---
title: "Git Force Push"
layout: post
date: 2018-07-25 17:17
image: /assets/images/markdown.jpg
headerImage: false
tag:
- git
- push
- bash
category: blog
author: carloschacon
description: "How to forcibly push your current Git branch"
---

## Introduction

This is my first post in this blog. I want to improve my writing skills, so I decided to start blogging about tech-related topics.
This time I will walk you through the thought process I went through to write a small Bash command to forcibly push the current branch upstream in Git.

## Context

Usually, when I'm developing a new feature or fixing a bug, either at work or in one my personal projects, I develop it on a feature branch, independent from
the master branch of the repo. Before, merging my changes to the master branch, I like to squash my commits into a single one to maintain a clean commit history.
Squashing the commits into a single one causes a conflict between my local branch and the upstream branch. Since I don't care about the changes upstream, what I like
to do, is to forcibly push my now-squashed changes upstream with a `git push -f origin MY_BRANCH` command.


This has become kind of tedious, it's a very repetitive process so I decided to write a small script to do it for me.

The things that I needed were:
1. Get my current branch.
2. Have a branch blacklist.
3. Add some coloring to make it pretty.
4. Force push the branch upstream.


Later on, I'll automate the squashing of the commits. I'll talk about it on another post.

## Current Branch

The first thing I need is to find out what is the current branch I'm working on. The simplest way to do it -that I know of, I'm not a Git hacker- is to use `git branch`
to get a list of all the branches.
The current branch is marked by an `*` next to the branch name. I used this to filter out the list with a `grep` command, like this:

```
git branch | grep "*"
```

I still cannot use the output of this command because it still includes the asterisk and some whitespace. To remove the unwanted characters, I used `awk` to clean the output:

```
git branch | grep "*" | awk '{print $2}'

```

Finally, I stored the end result in a handy variable that I will use later on:

```
current=`git branch | grep "*" | awk '{print $2}'`
```

## Branch Blacklist

Since running `git push -f origin BRANCH_NAME` is a destructive command, meaning that it will overwrite the upstream branch, I want to take some security measures.
Most of the branches are fine to be overwritten, however, there are some that it's better not to affect, such as `master` branch or the branch that production is using.
Therefore, we need some sort of blacklist to allow running the push command only if it's not one of these branches. Since this is the first pass of this tool, I will keep it
simple for now. A simple `if/else` statement that checks the branch name will do it:

```
  current=`git branch | grep "*" | awk '{print $2}'`;
  if [ $current = "master" ]; then
    echo 'This branch is forbidden!';
  else
    echo 'It is OK to push this branch!';
  fi
```

In this example, the `master` branch is the only forbidden branch. What happens if you want to restrict branches that start with a certain substring?
In this next example, I'll show you how to combine the previous example and checking a substring in the branch name:

```
  current=`git branch | grep "*" | awk '{print $2}'`;
  if [[ $current = *"production-"* || $current = "master" ]]; then
    echo 'This branch is forbidden!';
  else
    echo 'It is OK to push this branch!';
  fi
```

Supposing there are multiple branches with the prefix `production-`, this new addition will prevent the command to forcibly push to those branches, as well.

## Output Coloring

I like to have visual queues in the tools I work with. I thought adding some color to the output of this command would be nice, it will allow the user to know
if the command finished successfully or if there was an error.
I will add the branch name and color it green when the command succeeds and I'll color red the forbidden branch whenever it was not posible to push the branch.
Bash allows a certain degree of text formatting through the ANSI escape codes that I wasn't aware of. For now, I just need to add some colors but further styling
is posible.

First of all, I need to enable the parsing of escape sequences when I use `echo`.

```
echo -e "Escape sequences parsing enabled!"
```

Now, it's a matter a adding the ANSI codes of the colors and concatenating them to the output strings:

```
RED='\033[1;31m';
GREEN='\033[1;32m';
NOCOLOR='\033[0m';
current=`git branch | grep "*" | awk '{print $2}'`;
if [[ $current = *"production-"* || $current = "master" ]]; then
  echo -e "I can't force push ${RED}master${NOCOLOR} and ${RED}production${NOCOLOR} branches.\n";
else
  echo -e "Force pushing ${GREEN}$current${NOCOLOR} branch.\n";
fi

```


## Putting It All Together

We are almost done. The final piece to add is the `git push -f` command when the branch is allowed to be pushed:

```
RED='\033[1;31m';
GREEN='\033[1;32m';
NOCOLOR='\033[0m';
current=`git branch | grep "*" | awk '{print $2}'`;
if [[ $current = *"production-"* || $current = "master" ]]; then
  echo -e "I can't force push ${RED}master${NOCOLOR} and ${RED}production${NOCOLOR} branches.\n";
else
  echo -e "Force pushing ${GREEN}$current${NOCOLOR} branch.\n";
  git push -f origin $current;
fi
```

## Even More Security Measures

Instead of using `--force` or `-f`, we can use `--force-with-lease` to be more confident we will not break anything.
`force-with-lease` will only forcibly push our changes if the local and upstream branches are in-sync. If someone else pushed their changes, 
our push won't work because now our changes are out-of-date in relation to the upstream branch.


The final step is to wrap the script in a function for ease of use:


```
force_push() {
  RED='\033[1;31m';
  GREEN='\033[1;32m';
  NOCOLOR='\033[0m';
  current=`git branch | grep "*" | awk '{print $2}'`;
  if [[ $current = *"production-"* || $current = "master" ]]; then
    echo -e "I can't force push ${RED}master${NOCOLOR} and ${RED}production${NOCOLOR} branches.\n";
  else
    echo -e "Force pushing ${GREEN}$current${NOCOLOR} branch.\n";
    git push --force-with-lease origin $current;
  fi
}
```

