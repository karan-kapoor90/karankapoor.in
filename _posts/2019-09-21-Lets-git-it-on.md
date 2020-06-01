---
layout: post
categories: [Tutorials, Git]
comments: true
excerpt: A beginners guide to git
title: Lets Git it on
tags: [git, beginners, essentials]
description: A beginners guide to git.
---

While I was *trying* to learn how to use git more efficiently, I realised that it helped to have a handy guidebook. Here's some of the git commands that helped me along the way.. Let's "Git" it on!! 

This is essentially intended to be a basic starter kit for using git.


# A Subtle background

You're a software development team lead and you've got a bunch of newbies working on a piece of code. They're going to do the following:

* Feed off the basic code you put in place
* Develop features
* Make loads of mistakes
* Hopefully, fix the mistakes
* Release new versions of the code

Here's the scenario:

All of this is going to:


>       Happen on the same piece of code

>           At the same time

>               While new things are moving..    


# Setting up a new Repository


1. In a folder that you want to create a git repo out of, do the following:

    ``` bash
    $ git init    # Initiates a repository here
    ```


    >       This creates a branch called 'master'


2. Let's create your first file


    ``` bash
    $ echo "This is your first file" > README.md
    ```


    >     Git has the concept of a local staging area. When you want to add something to an existing checkpoint, it is compared with this Staging first before you commit it to the history (or state). But more on that later.


3. At this point one can check the status of the file with regards to the current state of the repository. Since your repo is a new one at this point, there's only a single *Untracked* file called README.md. 


    ``` bash
    $ git status
    On branch master    #You are on a branch called master

    No commits yet  #you haven't committed anything yet

    Untracked files:
      (use "git add <file>..." to include in what will be committed)
            README.MD

    nothing added to commit but untracked files present (use "git add" to track)    #self explainatory much?
    ```


4. We want to add this to the stage area we mentioned earlier.


    ``` bash
    $ git add . #add everything in the current location to my staging area
    $ git status
    On branch master
    No commits yet

    Changes to be committed:
      (use "git rm --cached <file>..." to unstage)
            new file:   README.MD
    ```


5. Now the idea of creating a checkpoint is that firstly, the checkpoint remains local to your machine. This is available to only you at this moment and cannot be seen by others collaborating on the repository..... which is kind of the general idea right??


    ``` bash
    $ git commit -m "Happy commit"    # not adding a commit message will throw an error. It's good manners really :)
    [master (root-commit) 2b3eb1b] Happy commit
    1 file changed, 1 insertion(+)
    create mode 100644 README.MD

    $ git status
    On branch master
    nothing to commit, working tree clean   #Just the way it should be
    ```


6. Now you've created a checkpoint of your work so far. At this point, you could either continue adding more stuff, staging and then commiting it, but the eventual goal is merging it into the parent repository. **Singularity!!**


    ``` bash
    $ git push  #because I want to push it to the git server
    fatal: No configured push destination.
    Either specify the URL from the command-line or configure a remote repository using

        git remote add <name> <url>

    and then push using the remote name

        git push <name>
    ```


    >     And we've hit another holy grail git concept. The error above says that no push destination is configured. A destination in this context is a git repository configured on a git server somewhere. One of the most popular free (and we hope it stays that way after Microsoft's acquisition) is github. Github or your own private repo, the intent is to collaborate and version control your code/ data!


7. Login to github.com using some social login or using your email ID and create a profile. And create an _empty_ repository called test. 


8. Once created, your repo would be at a link like `https://github.com/karan-kapoor90/test` and the actual git repository will be at `https://github.com/karan-kapoor90/test.git`. This the is "remote" server we want to push our code to. 
You have the option of talking to the repo using ssh or through https. ssh requires you to have the private key to your repo available locally so you don't have to punch in the password everytime. With https, it'll ask you for the password and username.


    ``` bash 
    $ git remote add origin https://github.com/karan-kapoor90/test.git  # adding a remote called origin to our github repo
    $ git push origin master    # since our first branch is called master and the remote name is origin

    # Enter your credentials if requested.
    Enumerating objects: 3, done.
    Counting objects: 100% (3/3), done.
    Writing objects: 100% (3/3), 239 bytes | 39.00 KiB/s, done.
    Total 3 (delta 0), reused 0 (delta 0)
    To https://github.com/karan-kapoor90/test.git
    * [new branch]      master -> master
    ```


8. Voila! Your first git repo is up and running!!


# Be a team player - Starting with existing code


As a member of a highly efficient team working to wow your customers with the awesome stuff you create, you're going to start somewhere by getting a copy of the base code. Depending on what what you do in the team, you might use this code to test it, develop etc. 


1. The satrting point is a base code repository that you have to checkout and start working on. From the previous section, you might remember that the primary brach is called the master branch. 


    >       There are multiple ways of doing this:
    >           Git Pull: A shorthand for fetching a remote branch and merging it into your current local branch
    >           Git Fetch and Git Checkout: In succession, this will pull the remote branch you ask for from the remote and switch to that branch locally. Meaning any changes you do after this would be in the new branch you've just checked in.
    >           Git Clone: For the first time you copy a remote repository to your machine. Sets up the local repository, the origin, branches etc. locally.

  
    ``` bash 
    $ git clone https://github.com/karan-kapoor90/test.git # the repository 
    Cloning into 'test'...
    remote: Enumerating objects: 3, done.
    remote: Counting objects: 100% (3/3), done.
    remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
    Unpacking objects: 100% (3/3), done.
    ```


    >       When you clone for the first time, you essentially add the repository URL as a remote called 'origin' as well.


2. At this point, at your current location, a folder by the name of the repository ('test' in this case) is created. When you get into the folder, you'll be able to see the contents of the online repository in the folder. 


    ``` bash
    $ cd test # go into the test folder
    $ git status
    On branch master    # Note that you're on the branch called master
    Your branch is up to date with 'origin/master'.

    nothing to commit, working tree clean
    ```


3. While you now have your code, you can start working on the code. Ideally though, you should create a new branch here and code your features in that branch. This way you can track the feature you're building, push it to the repository for checkpointing, automated testing etc. This is typically the best practice for developing features independent of the main code. 


    ``` bash
    $ git branch newfeature # where newfeature is the name of the new branch
    $ git checkout newfeature    # to switch to the newly created branch
    ```

    or, you could perform both the steps together using the command (this is **instead** of the 2 steps metioned above):

    ``` bash
    $ git checkout -b newfeature     # creates a new branch called newfeature and switches to this branch
    ```

4. From this point on, all the previously uncommitted changes move to the newfeature branch you just created. 


    ``` bash
    $ git status    # unstaged changes from when you moved to a new branch
    ```


5. From here on, the process of committing and getting your code to the remote repository is pretty much the same as before except one tiny difference.


    ``` bash
    $ git add .   # Add changed files to the stage
    $ git commit -m "Added content to the file"     # Creating a local commit
    ```


6. The tiny change, is that the remote repository doesn't know (and doesn't have it) of this new branch


    ``` bash
    $ git push origin newfeature    # origin is the remote name; newfeature is the branch
    ```


And that's it for Part 1, basics of working with git! Coming up with more soon ;-)
