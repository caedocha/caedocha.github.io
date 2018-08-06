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

For my first post, I will walk you through the process of writing a small Bash script to force push a Git branch upstream.

## Context

I use feature branches, independent from the master branch, to work on a new feature or to fix a bug.
I like to squash my commits into a single one before merging them to the master branch to keep the commit history clean.
However, this causes a conflict between my local branch and the upstream one. I can force push my local changes to overwrite 
the upstream branch with `git push -f origin MY_FEATURE_BRANCH`.

This has become a very repetitive and tedious process so I decided to write a small script to automate it.

The things that I need are:
1. Get my current branch.
2. Have a branch blacklist.
3. Add some coloring to make it look pretty.
4. Force push the branch upstream.


Later on, I will automate the commit squashing. I will write about it in my next post.

## Current Branch

The first thing I would need to do is find out what the current branch is. The simplest way to do it (that I know of, I'm not a Git hacker) is to use `git branch`
to get a list of all the branches.
The current branch is marked by an `*` next to the branch name. I used this to filter out the list with a `grep` command, like this:

```
git branch | grep "*"
```

I still cannot use the output of this command because it includes the asterisk and some whitespace. To remove the unwanted characters, I used `awk` to clean the output:

```
git branch | grep "*" | awk '{print $2}'

```

Finally, I stored the end result in a handy variable that I will use later on:

```
current=`git branch | grep "*" | awk '{print $2}'`
```

## Branch Blacklist

Since running `git push -f origin BRANCH_NAME` is a destructive command, meaning that it will overwrite the upstream branch, I want to take some security measures.
While most branches can be overwritten without issues, it would be wise to prevent overwriting important branches such as master or production.
Therefore, we need some sort of blacklist to allow running the push command only if it is not one of these branches. Since this is the first pass of this tool, I will keep it
simple for now. A simple `if/else` statement that checks the branch name will do the trick:

```
  current=`git branch | grep "*" | awk '{print $2}'`;
  if [ $current = "master" ]; then
    echo 'This branch is forbidden!';
  else
    echo 'It is OK to push this branch!';
  fi
```

In this example, the `master` branch is the only forbidden branch. What happens if you want to restrict branches that start with a certain substring?
In this next example, I will show you how to combine the previous example and checking a substring in the branch name:

```
  current=`git branch | grep "*" | awk '{print $2}'`;
  if [[ $current = *"production-"* || $current = "master" ]]; then
    echo 'This branch is forbidden!';
  else
    echo 'It is OK to push this branch!';
  fi
```

Assuming there are multiple branches with the prefix `production-`, this new addition will prevent the command to forcibly push to those branches as well.

## Output Coloring

I like to have visual queues in the tools I work with. So I thought adding some color to the output of the script would be helpful.
When the script pushes the branch successfully, I will display a feedback message in bright green including the branch name.
When the script tries to push a blacklisted branch, I will display a feedback message in bright red.
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
`force-with-lease` will only force push our changes if the local and upstream branches are in-sync. If someone else pushed their changes to that branch,
our push won't work because our branch is out-of-date in relation to the upstream branch.

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

