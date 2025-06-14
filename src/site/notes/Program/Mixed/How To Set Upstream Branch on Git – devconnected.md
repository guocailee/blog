---
{"dg-publish":true,"permalink":"/Program/Mixed/How To Set Upstream Branch on Git – devconnected/","noteIcon":"","created":"2025-03-06T21:28:25.978+08:00"}
---

#Git 

When cloning a Git repository or creating new feature branches, you will have **to set upstream branches** in order to work properly.

_But what are upstream branches?_

Upstream branches are closely associated with remote branches.

**Upstream branches define the branch tracked on the remote repository by your local remote branch (also called the remote tracking branch)**

![](https://devconnected.com/wp-content/uploads/2019/10/upstream-1-1024x702.png)

When creating a new branch, or when working with existing branches, it can be quite useful to know **how you can set upstream branches on Git.**

Table of Contents

-   [Set upstream branch using git push](#Set_upstream_branch_using_git_push "Set upstream branch using git push")
-   [Set upstream branch using an alias](#Set_upstream_branch_using_an_alias "Set upstream branch using an alias")
    -   [Using a git alias](#Using_a_git_alias "Using a git alias")
    -   [Using a bash alias](#Using_a_bash_alias "Using a bash alias")
-   [Set upstream branch for an existing remote branch](#Set_upstream_branch_for_an_existing_remote_branch "Set upstream branch for an existing remote branch")
    -   [Set tracking branches for new local branches](#Set_tracking_branches_for_new_local_branches "Set tracking branches for new local branches")
    -   [Set tracking branches for existing local branches](#Set_tracking_branches_for_existing_local_branches "Set tracking branches for existing local branches")
-   [Why are upstream branches so useful in Git?](#Why_are_upstream_branches_so_useful_in_Git "Why are upstream branches so useful in Git?")
-   [Inspecting tracking branches configuration](#Inspecting_tracking_branches_configuration "Inspecting tracking branches configuration")
-   [Conclusion](#Conclusion "Conclusion")

## Set upstream branch using git push

**The easiest way to set the upstream branch is to use the “[git push](https://git-scm.com/docs/git-push)” command with the “-u” option for upstream branch.**

```bash
$ git push -u <remote> <branch>
```

Alternatively, you can use the “**–set-upstream**” option that is equivalent to the “-u” option.

```bash
$ git push --set-upstream <remote> <branch>
```

As an example, let’s say that you created a branch named “**branch**” using the checkout command.

```bash
$ git checkout -b branch
Switched to a new branch 'branch'
```

You can check tracking branches by running the “**git branch**” command with the “**-vv**” option.

```bash
$ git branch -vv
* branch  808b598 Initial commit
 master  808b598 [origin/master] Initial commit
```

As you can see, compared to master, the branch “branch” has no tracking branches yet (and no upstream branches as a consequence)

**We can set the upstream branch using the “git push” command.**

```bash
$ git push -u origin branch
Total 0 (delta 0), reused 0 (delta 0)
 * [new branch]      branch -> branch
Branch 'branch' set up to track remote branch 'branch' from 'origin'.
```

Let’s have a look at the tracking branches again with the branch command.

```bash
$ git branch -vv
* branch  808b598 [origin/branch] Initial commit
  master  808b598 [origin/master] Initial commit
```

Great!

We have successfully set the upstream branch for our newly created branch.

## Set upstream branch using an alias

**Another way to set the upstream branch is to define an alias for your “git push” command.**

In fact, pushing to HEAD is equivalent to pushing to a remote branch having the same name as your current branch.

```bash
$ git push -u origin HEAD
```

In order to avoid having to define the upstream everytime you create a new branch, define an alias for the command we just wrote.

For aliases, you have two choices, you can either create a git alias or a bash alias.

### Using a git alias

In order to create a new git alias, use the “**git config**” command and define a new alias named “pushd”

```bash
$ git config --global alias.pushd "push -u origin HEAD"
```

When you are done adding and committing fiels to your repository, set the upstream branch using your newly defined alias.

```bash
$ git pushd
Total 0 (delta 0), reused 0 (delta 0)
 * [new branch]      HEAD -> branch
Branch 'branch' set up to track remote branch 'branch' from 'origin'.
```

### Using a bash alias

Alternatively, you can use a bash alias if you don’t want to modify your existing git commands.

Define a **new bash alias** using the “**alias**” command and define a name for it.

```bash
$ alias gp='git push -u origin HEAD'
```

Let’s create a new branch and use our alias in order to push our code and create the upstream branch easily.

```bash
$ git checkout -b branch2
Total 0 (delta 0), reused 0 (delta 0)
 * [new branch]      HEAD -> branch2
Branch 'branch2' set up to track remote branch 'branch2' from 'origin'.
```

## Set upstream branch for an existing remote branch

In some cases, you may choose to link your local branches to existing remote branches that you just pulled or [cloned from the main repository](https://devconnected.com/how-to-clone-a-git-repository/).

Let’s say for example that you pulled the “dev” branch located on the “origin” remote.

As a consequence, the tracking branch is named “origin/dev”.

### Set tracking branches for new local branches

**In order to switch to the local “dev” branch, and to set the “origin/dev” as the tracking branch (or upstream branch), use the “–track” option.**

```bash
$ git checkout --track origin/dev

Branch 'dev' set up to track remote branch 'dev' from 'origin'.
Switched to a new branch 'dev'
```

To verify that you linked dev to the tracking branch “origin/dev” (which upstream branch is the remote dev), use the “[git branch](https://git-scm.com/docs/git-branch)” command.

```bash
$ git branch -vv
* dev 808b598 [origin/dev] Initial commit
```

### Set tracking branches for existing local branches

On the other hand, you may have chosen to work on a local branch and to set the upstream branch (or the remote tracking branch later on).

**It is perfectly fine, but you will have to use the “git branch” in order to set the existing branch upstream branch.**

```bash
$ git branch -u <remote>/<branch>
```

Let’s take the example of the “feature” branch that you just created to start working.

```bash
$ git checkout -b feature
Switched to a new branch 'feature'
```

You created some commits in your branch, you want to set the tracking branch to be master.

```bash
$ git branch -u origin/master
Branch 'feature' set up to track remote branch 'master' from 'origin'.
```

Great! You successfully set the upstream branch for your existing local branch.

## Why are upstream branches so useful in Git?

Upstream branches are useful because :

-   **You get references to your remote repositories and you essentially know if you are ahead of them or not.**

When performing a “git fetch” command, you can bring the new commits from your remote repository and you can choose to merge them at will.

-   **You can perform pull and push easily**

When you set your upstream (or tracking) branches, you can simply execute pulls and pushes without having to specify the target branch.

Git automatically knows that it has to fetch the new commits to the remote tracking branch. Similarly, Git already knows that it has to push new commits to the upstream branch.

But where does Git keep a reference of the upstream branches associated with local branches?

Git keeps references to upstream branches via its config file in the “.git” directory.

## Inspecting tracking branches configuration

In order to inspect your current Git configuration, [list the hidden files and directories](https://devconnected.com/how-to-show-hidden-files-on-linux/) in your current working Git directory.

```bash
$ ls -al

total 16
drwxrwxr-x 3 schkn schkn 4096 Nov  5 16:10 .
drwxrwxr-x 7 schkn schkn 4096 Nov  5 16:10 ..
drwxrwxr-x 8 schkn schkn 4096 Nov  6 10:27 .git
```

Now, inspect the content of the “config” file located in the .git directory.

```bash
$ cat .git/config

[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = <repo_url>
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
```

As you can see, Git keeps a reference between your local branch, the name of the remote and the branch it has to merge with.

## Conclusion

In this tutorial, you learnt more about upstream branches and how they are related to remote tracking branches in Git.

You learnt different techniques in order to set remote tracking branches using a command or an alias to set it.

You also learnt how you can link your current local branches to existing remote tracking branches easily with the branch command.

If you are interested in Software Engineering, we have a complete section dedicated to it on the website so make sure to have a look.

 [https://devconnected.com/how-to-set-upstream-branch-on-git/](https://devconnected.com/how-to-set-upstream-branch-on-git/)
