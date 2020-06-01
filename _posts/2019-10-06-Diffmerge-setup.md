---
layout: post
categories: [Tutorials, Git]
comments: true
excerpt: Diffmerge Setup guide for windows
title: Efficient Git Merge - Diffmerge
tags: [git, beginners, essentials, Diffmerge]
description: Diffmerge Setup guide for windows
---

Imagine your team and you have been working on loads of code and it's time to bring it all together. This is one of those dreaded (read "group collaboration") times when the developers and the leads typically sit together and start merging code, fingers crossed hoping to get it right in as little iterations as possible.

DiffMerge is a free tool used to resolve merges on a repository. It helps you do a side-by-side comparison of each file that has diffs. For example say you have a repository at a commit A (on branch master). You create a new branch of commit A and make some changes to it. Then you push it as commit B to a new-branch. While you're working on the new-branch on commit B, somebody updated the same files in the master and got it to a commit C. When you try to merge your changes upstream into the master branch, if both commit C and your commit have made changes to the same files, a merge conflict will arise. This conflict can be resolved using this diffMerge tool. The tool is freeware and allows for easy 2-way and 3-way merges.

***Full disclosure***, I've tested this on windows 7 and 10 only. AT the time of this writing in late 2019, the Diffmerge is at version 4.2.0.697.stable.

# Setup

## Download & Install Diffmerge


1. Navigate to [Diffmerge Download](https://sourcegear.com/diffmerge/downloads.php) and download diffmerge for your platform. 
2. Run the setup and install the software into your local machine. I wasn't able to change the directory it installs into because the Browse button was greyed out, but it's a relatively lightweight app so it won't be a space guzzler on your primary drive either. 
3. To double check the installation, you can open a Git Bash, cmd or a PowerShell Session and type the command `sgdm`. This should open an empty DiffMerge window.


## Configuring Git


Now that you've got Diffmerge installed and working, using some of the pre-defined git properties, we can setup Diffmerge to be the default diff and merge tool.

>       Some of the configurations are specific to the terminal you use on windows. This tutorial configuration assumes you're using git bash. For other git distributions/ terminals, [see here](https://sourcegear.com/diffmerge/webhelp/sec__git__windows.html) .

The following commands would suffice:

```bash
# To configure diffmerge as the default "Difftool"
$ git config --global diff.tool diffmerge
$ git config --global difftool.diffmerge.cmd "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm_cygwin.sh -p1=\"\$LOCAL\" -p2=\"\$REMOTE\" --title1="Original" --title2="Modified""

# To configure diffmerge as the default "mergetool"
$ git config --global merge.tool diffmerge
$ git config --global mergetool.diffmerge.cmd "C:/Program\ Files/SourceGear/Common/DiffMerge/sgdm_cygwin.sh -merge -result=\"\$MERGED\" -p1=\"\$LOCAL\" -p2=\"\$BASE\" -p3=\"\$REMOTE\" --title1="CurrentBranch" --title2="Result" --title3="IncomingBranch""
$ git config --global mergetool.diffmerge.trustExitCode true    # Git should trust the merge exit code returned by the mergetool
$ git config --global mergetool.keepBackup false    # Do not keep the .orig backup file post merge
```

# Usage

## Background

You've got a respository called Test with a master branch and a few commits.


<div class="mermaid text-center">
graph TB
    subgraph master
    baseCommit-->layout-->profile
    end
style master fill:#f1f1f1
style master stroke-width:0px
</div>


Next thing you create a new branch to create a search feature. Make some code changes, stage them and commit them (just like excellent developers do :) ).


<div class="mermaid text-center">
graph TB
    subgraph master
    m1[baseCommit] --> m2[layout] --> m3[profile]
    end

    subgraph searchBranch
    s1[baseCommit] --> s2[layout] --> s3[profile] --> s4[search1.0]
    end
style master fill:#f1f1f1
style master stroke-width:0px
style searchBranch fill:#f1f1f1
style searchBranch stroke-width:0px
</div>

Suddenly you're told you need to integrate search for Usernames that's gone live in the master branch!! That basically means:

<div class="mermaid text-center">
graph TD
    id1(Switch to the master branch) --> id2(Get the latest code) --> id3(Switch over to your new branch) --> id4(Merge the latest code into your branch) --> id5(Run mergetool)-->id6(Add and commit changes)
</div>


The current state should look something like:


<div class="mermaid text-center">
graph TB
    subgraph master
    m1[baseCommit]-->m2[layout]-->m3[profile]-->m4[usernames]
    end
    subgraph searchBranch
    s1[baseCommit]-->s2[layout]-->s3[profile]-->s4[search1.0]
    end
style master fill:#f1f1f1
style master stroke-width:0px
style searchBranch fill:#f1f1f1
style searchBranch stroke-width:0px
</div>


And since you only want to merge the code into your branch and not yet push the search feature to master, the end state should look like:

<div class="mermaid text-center">
graph TB
    subgraph master
    m1[baseCommit]-->m2[layout]-->m3[profile]-->m4[usernames]
    end
    subgraph searchBranch
    s1[baseCommit]-->s2[layout]-->s3[profile]-->s4[search1.0]-->s5[usernames]-->s6[Merge branch 'master' into searchBranch]
    end
style master fill:#f1f1f1
style master stroke-width:0px
style searchBranch fill:#f1f1f1
style searchBranch stroke-width:0px
</div>


## HowTo:

1. You're on master branch
    ```bash
    $ git checkout master
    ```

2. Creating a new branch called searchBranch

    ```bash
    $ git checkout -b searchBranch
    # You create the super awesome search functionality, the next best thing to google perhaps?
    $ git add .
    $ git commit -m "search 0.1"
    ```
3. Getting the latest "username" code on your local master branch

    ```bash
    $ git checkout master
    $ git pull
    ```

    > A merge conflict arises only if there have been changes on the same file in the 2 codebases you're trying to merge. Files that don't have changes in both codebases are automerged by git! Git is smart :-)


4. Merging into the search branch

    ```bash
    $ git merge master

    Auto-merging <filename>
    CONFLICT (content): Merge conflict in README.MD
    Automatic merge failed; fix conflicts and then commit the result.
    ```
5. Mergetool to the rescue!!

    ```bash
    $ git mergetool
    ```
6. Move the final code (what the fixed code should look like) into the middle quadrant, save the file and exit the mergetool.
7. This will close the merge session once you've done it for all the files that required a manual merging and create a new commit called Merge branch 'master' into searchBranch. Use `$git log` to view the commit history on the branch. Interestingly, you'll notice a commit called username followed by a commit named Merge branch 'master' into searchBranch. The reason git does this is because it tracks history closely and ensures that you can always track what point changes were made to the codebase.  

By the way, it's common practice to delete the branch you were developing the new feature once it's been merged into master and has served it's purpose. 

```bash
# Remove the local copy of the branch
$ git branch -D searchFeature

# Remove the remote copy/ branch from the server
$ git push :searchFeature   # the ":" tells the remote server to delete the branch
```


> But what if the changes I was making to the search branch weren't commit ready when the username requirement came? That's a story for another blog, Git Stash to the rescue.