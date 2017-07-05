---
title: Getting into git & github - a basic dev workflow
tags: tag1, tag2
category: code
author: Lukas Oppermann
preview: An easy to understand instruction to get you up an running with git and github in a matter of minutes.
description: Get started with using git and github to improve your development workflow
---

> Getting everything ready to start development can be scary, but if you take small steps it is actually quite easy.

## The idea behind GIT & github
While not a necessity, I strongly recommend using GIT and github to track your version history. [GIT](https://git-scm.com/) is a free version control system. This means you can incrementally save the changes you to a document. The benefit: highly precise version control and minimal size. [GitHub](http://github.com) is an only platform to remotely store your projects. You can use GIT only locally, but a remote storage has he benefit of being available from everywhere, if you wipe your computer, your projects are safe and sound in the cloud. Github allows for unlimited public projects, you only have to pay if you want to have private projects.

## Get GIT up an running
I am using a mac, you should have git preinstalled in your mac. You can test this by running `git --version` from the Terminal.app. You might have to accept and install the xCode command line tools, this is fine, just do it. **If you do not want to have control over your git version, your are done! Go to the next section.**

However I personally like to be in control of which GIT version I am running, because sometimes you want a feature, but apple does not update git very frequently. To install GIT on your mac, the easiest way is using [Homebrew](http://brew.sh/), just run the following command in your terminal to install homebrew to your mac.

```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```


To install GIT using homebrew, run the following command in your terminal.

```bash
brew install git
```

If you run `git --version` on your terminal you still get your old GIT version, because you need to tell your mac where to find the correct GIT version before it will be used.
You need to edit or create your `.bash_profile` file and add `/usr/local/bin` to the `PATH`.

```bash
# try to open the .bash_profile
open ~/.bash_profile
# if you have no bash_profile yet, you will get the following message
# > The file /Users/yourUserName/.bash_profile does not exist.

# Run this command to create the file
touch ~/.bash_profile
# now open it
open ~/.bash_profile
```

In your `.bash_profile` add (or edit) the `export PATH` line. The `PATH` variable is basically a line of paths, separated by colons `:` which at least to me is pretty confusing, but thats the way it works. The importance is from left to right, so now we first look inside `/usr/local/bin` before we look anywhere else.

```bash
# add this to your .bash_profile file
export PATH="/usr/local/bin:$PATH"
```
After closing and reopening your command line, `which git` should show the correct path `/usr/local/bin/git` and `git --version` should show the correct version.

## Set up your GIT user
Once you have GIT ready, add your global GIT email and username, which will be used to track which changes you committed. In your terminal enter the following to configure your GIT. By using the `--global` flag the settings are used for all your projects.

```bash
git config --global user.name "YOUR NAME"
git config --global user.email "YOUR EMAIL ADDRESS"
```

You can view the settings by typing the following into your terminal. To change it, just use the command from above.

```bash
git config --global --get user.name
git config --global --get user.email
```

## Getting started with github
First, of course, if you want to use github.com you need to [sign up](https://github.com/join), go for a free account if you have no reason not to, you can always upgrade.

The easiest way to connect to github is by using *HTTPS*, while ssh is possible, it is more complicated. To do this, first we need to [create a new project](https://github.com/new), in the GIT world projects are called *repositories* or just *repos*. Do this by clicking the plus symbol in the top right corner and selecting *new repository* from the dropdown.

Now you can *clone down* the repo, the url will be displayed on the new repository page. Make sure you grab the *HTTPS* url. This page also has a quick tutorial on how to create a new repo using the terminal / command line.

```bash
# make sure you move to your code folder, however you named it
cd ~/Code
# clone the repo, you can provide the folder name as an argument
git clone https://github.com/username/repository.git name-of-the-new-folder
```

Now your repository is ready to use. Basic GIT usage is pretty simple...

## GIT basics
Now that we have GIT set up and github connected, we can dive into some git basics. First, whenever you want to know how your repository is doing, you can just type `git status`. At the moment there is nothing in our repo, so lets create a file to commit to our GIT. Move to your terminal app and enter the following.

```bash
# the .gitignore file, describes which files are supposed to be ignored
touch .gitignore
# use git add [filename] to add a file to be tracked by your GIT
# untracked files will be listed as untracked, but changes can not be saved / restored
git add .gitignore
# add a commit message, so you will know what you were about, later on
git commit -m 'adding an empty gitignore file'
# get the status of your repo
git status
# you should see this, it means that all local changes are committed to your version history locally
> On branch master
> Your branch is ahead of 'origin/master' by 1 commit.
>   (use "git push" to publish your local commits)
> nothing to commit, working directory clean
```

Since you are working with a repo on github, you most likely want to store your version history in the cloud. Since your repository is already connected to github, this is as simple as a `git push`.

```bash
# upload your committed changes to your remote (online) repository
git push
# if you never pushed to github before you will be asked for your github username & password
# the password will not be displayed
# you should see something similar to this
> Counting objects: 1, done.
> Delta compression using up to 8 threads.
> Compressing objects: 100% (1/1), done.
> Writing objects: 100% (1/1), 263 bytes | 0 bytes/s, done.
> Total 1 (delta 0), reused 0 (delta 0)
> To https://github.com/username/repository.git
   6d2c3d1..4784692  master -> master
```

This is it, if you look at your repository on github you will see the `.gitignore` file is now present. So you are done. Whenever you feel you want to save your changes, you just need to `git add` the files you want to save and `git commit` the changes, including a message that will make it possible for you to remember what you changed / added. Lets try this out by editing and committing the `.gitignore` file, if you did not enable invisible files to be visible yet, just open it via the terminal.

```bash
open .gitignore
```

Now just add all files and directories you do **NOT** want to be tracked. One per line. Save and move back to the terminal app.

```bash
.DS_Store
folderName/
```

Before we commit the changes, lets quickly add an empty `readme.md` file. This will show up on your repositories homepage on github.

```bash
# add an empty readme file
touch readme.md
# check the status of the git repository
git status
# you should see something like
> On branch master
> Your branch is up-to-date with 'origin/master'.
> Changes not staged for commit:
>  (use "git add <file>..." to update what will be committed)
>  (use "git checkout -- <file>..." to discard changes in working directory)
>
>	modified:   .gitignore
>
> Untracked files:
>  (use "git add <file>..." to include in what will be committed)
>
>	readme.md
```

Now we need to repeat what we did before: *add – commit – push*. There is a way to add all files to your commit, `git add --all`, but *beware*, because you can easily add unwanted files to your repository. If in doubt, use `git add filename`.

```bash
# you can add multiple files at once by separating them with a space
git add .gitignore readme.md
# commit files & add a commit message
git commit -m 'update gitignore and add empty readme.md'
# push to remote
git push
```

## Store your password
Because it is pretty annoying to enter your username and password every time you want to push something to github, there is a tool to deal with it. Git has a way of storing your credentials in the OSX keychain, so that you do not have to enter them manually.

If you installed git via homebrew it should already be installed, you can test it by entering the following command into the terminal.

```bash
git credential-osxkeychain
# output should be
> Usage: git credential-osxkeychain <get|store|erase>
# if you get the following it is not installed
> git: 'credential-osxkeychain' is not a git command. See 'git --help'.
```

To install `git credential-osxkeychain` use the following curl command in your terminal. Afterwards you will need to adjust the permissions, move the file to the correct directory (same as your git installation) and set git to always use the helper.

```bash
curl -s -O https://github-media-downloads.s3.amazonaws.com/osx/git-credential-osxkeychain
# adjust permissions
chmod u+x git-credential-osxkeychain
# move the git credential-osxkeychain to the path where git is installed
sudo mv git-credential-osxkeychain "$(dirname $(which git))/git-credential-osxkeychain"
# it will as for your password
Password: [enter your password]
# set git to use the osxkeychain credential helper
git config --global credential.helper osxkeychain
```

Done. Okay, this was a little more work, but you only need to do it once. The next time you push, git will as you for the username and password and to allow the keychain to save those. Afterwards, no password ever again.

## GIT is easy
So this it all you need for now. There is a lot more you can go with GIT and I will be writing some more articles about it. But if you understand the *add – commit – push* method you are good smaller projects, keep using it and onceit'ssecond nature, learning new things you can do with GIT will be easy.
