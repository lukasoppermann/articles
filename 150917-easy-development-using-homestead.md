---
title: Easy development using homestead
tags: tag1, tag2
author: Lukas Oppermann
category: code
preview: The why and how of using the homestead vagrant box. A virtual machine build by Taylor Otwell, the creator of Laravel. With homestead you have a plug and play virtual box for all your php projects.
description: Improve your development by using homestead the easiest, plug and play virtual machine for any php app.
---

> In this post we will setup homestead on your machine and get a Laravel installation up an running. Note, that while homestead is created by Taylor Otwell, the guy behind Laravel & Lumen, you can use it for any php project, no matter what framework you use.

## Why should I use homestead
The main benefit of homestead is that you have a complete setup (Nginx web server, PHP, MySQL, Postgres, Redis, Memcached, Node, etc.) in one installation. Additonally, homestead is maintained by a community so you do not have to worry about keeping it up to date yourself. Homestead checks for updates automatically and you just need to use it. It's a dev environment without the hassle and once get a new computer, getting your dev environment setup is as easy as following this tutorial.

## Preparing your machine

If you haven't been living under a rock the last couple of years you probably have most of the following things installed and configured already, but if you are new to the world of programming, chances are you are missing one or two of the dependencies. I will walk you trough the process assuming you have a clean installation of your OS an nothing else. I am working on a mac, if you are running linux or windows, things might be slightly different.

## VirtualBox & Vagrant
Homestead is a [Vagrant](https://www.vagrantup.com/downloads.html) virtual box, Vagrant is a virtual machine that runs on [VirtualBox](https://www.virtualbox.org/wiki/Downloads). You do not really need to know how it works, if you do want to know, check out their websites. What you need to know is that homestead runs a virtual server on your machine and mirrors everything from one specific folder to your virtual server.

First download and install [VirtualBox](https://www.virtualbox.org/wiki/Downloads). Just choose the version from the **VirtualBox platform packages** which corresponds to your operating system.

Once you have VirtualBox installed, download and install [Vagrant](https://www.vagrantup.com/downloads.html).

Now that both VirtualBox & Vagrant are installed on your machine, it's time to add homestead by simply running the following command in your command line. This may take a very long while...

```bash
vagrant box add laravel/homestead
```

While we wait, let's get everything else up and running.

## GIT
Git is already installed, but I prefer to have control over which version of git I am running, so I suggest to use your own installation. We are not going to actively use git, but git is used by vagrant so we need it. On OSX you can use [**homebrew**](http://brew.sh/), a dependency management tool for mac, to easily install & update git and other dependencies. You can install this using ruby, which comes preinstalled on every mac. Just run the following on your command line to install homebrew and git.

```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install git
```

To get your mac to actually the homebrew git version, you need to edit or create your `.bash_profile` file and add `/usr/local/bin` to the `PATH` after reopening your command line, `which git` should show the correct path and `git --version` should show the correct version.

```bash
# if you have no bash_profile yet
touch ~/.bash_profile
# now open it
open ~/.bash_profile
```

In your `.bash_profile` add (or edit) the `export PATH` line. The `PATH` variable is basically a line of paths, separated by colons `:` which at least to me is pretty confusing, but thats the way it works. The importance is from left to right, so now we first look inside `/usr/local/bin` before we look anywhere else.

```bash
export PATH="/usr/local/bin:$PATH"
```

## Installing composer
Having Git setup we can move on to [composer](http://www.getcomposer.org), which is the PHP dependency management tool. We will use this to create a new Laravel project and install the Homestead CLI. No matter what framework you use, composer is your friend for install your packages & dependencies.

```bash
# download & install composer
curl -sS https://getcomposer.org/installer | php
## move composer phar so that you can use just with composer
mv composer.phar /usr/local/bin/composer
```

If moving `composer.phar` fails due to permissions, run the command with `sudo`.

```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

Afterwards you need to adjust your `PATH` in the `.bash_profile` file, like we when installing git. This time we need to add composers global directory.

```bash
export PATH="~/.composer/vendor/bin:/usr/local/bin:$PATH"
# reload your command line
source ~/.bash_profile
```

## Install homestead globally

Homestead has a very handy command line tool, which we can install via composer. We will use this tool to edit the settings of our vm and start it up (you will need to restart your vm every time you restart your computer).

```bash
composer global require "laravel/homestead=~2.0"
```

## Creating your project

Now let us create a new project. If you do not already have a `Code` folder create one and `cd` into it. Create a new project using the `composer create-project` command, the last argument in the example is the name of the folder which will be created for your project, in this case it is `myApp`. Once this is done you should have the new folder in your `Code` directory.

```bash
mkdir ~/Code
cd ~/Code
composer create-project laravel/laravel --prefer-dist myApp
```

## Creating your ssh keys
If you do not have a public/private key pair on your machine, you will need to create one. Those keys are used by homestead to authorize itself to the virtual server when transferring files. Run the following command in your command line, when asked to provide a file in which to save the key, the default `/Users/yourUser/.ssh/id_rsa` is fine, just confirm by pressing the `enter` key. Afterwards you will be asked for a passphrase, choose a secure passphrase you can remember and confirm with `enter`. Afterwards reenter the passphrase and confirm again. Note that the command line will not show your password (or any characters like \* but just a blank space).

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@domain.com"

# output will be similar to
Your identification has been saved in /Users/yourUser/.ssh/id_rsa.
Your public key has been saved in /Users/yourUser/.ssh/id_rsa.pub.
The key fingerprint is:
05:f3:41:3b:ca:85:d6:17:a1:7d:f0:68:9d:f0:a2:db your_email@domain.com
```

Now you can add the key to your `ssh-agent` so you do not have to enter your passphrase all the time. Just run the following in the command line.

```bash
# test that ssh-agent is runnung
eval "$(ssh-agent -s)" # outputs something like; Agent pid 57640

# add your ssh key
ssh-add ~/.ssh/id_rsa
```

## Adding your first project url to homestead
Once the homestead box is installed (hopefully it is done by now), we can move on. We want to locally access `http://myapp.dev` in the browser and view our app. To achieve this we need to make our mac process all requests to this url locally, which we can do by pointing this url to the ip `192.168.10.10` in the `/etc/hosts` file. Homestead will pick up the requests and if we configure it correctly in the `homestead.yaml` point all requests to a folder we specify. Run `open /etc/hosts` in your command line and add the following line to it.

```bash
192.168.10.10  myapp.dev
```

**Update:** if you are on OSX 10.11 this will not work due to the *rootless* feature. You need to edit the file using nano. To do so, run `sudo nano /etc/hosts` from your terminal. Now you can add the line above and press `ctrl + x` when you are done. Type `y` and hit `enter` when asked if you want to save.

Should you get an error due to missing permissions, use vim to edit the file. In vim you navigate with the keyboard arrow keys. To go into "insert mode" and change the file, press the `i` key. Once you aer done press `Esc` to exit insert mode and type `:qw` and hit the `enter` key to save and close the file (you can close a file without saving by simply typing `:q`). Now we need to init homestead and edit the `homestead.yaml` file. You can probably leave most settings as they are, but I will run you through some of them nevertheless. Run the following two commands on your command line.

```bash
homestead init
homestead edit
```

- **ip** is the ip homestead listens on, this is must be the same as in your hosts file.
- **authorize** & **keys** are your ssh credentials, which homestead uses to mirror your files to the virtual server.
- **folders** defines the folders that are mirrored to the virtual server, I recommend using the default of `Code` as your folder, so you do not have to change this setting every time you install homestead. `map` is the folder on your machine and `to` the folder on the virtual server.
- **sites** defines which url will point to which folder on the virtual server.
- **databases** are the databases homestead will create whenever it is started, note that no migrations are run,it'sjust an empty mysql database.
- **variables** are environment variables that are set on the virtual server.

The only change we have to make in this file whenever we add a new project is to add a new entry in the `sites` section, like the one shown below. We create a Laravel project in a folder named `myApp`. The `index.php` file for every Laravel project is always within a `public` folder so we have to point our domain to `myApp/public` on our virtual server. If you did not change the `folders` section, the `map` there should point to `/home/vagrant/Code`, we need to prepend this to our project folder. This results in the path `/home/vagrant/Code/myApp/public`.

```bash
sites:
    - map: myapp.dev
      to: /home/vagrant/Code/myApp/public
```

Every time you add a new site to homestead you need to run `homestead destroy` to terminate the virtual server and `homestead up` to start it back up again. **Be aware** that this deletes all your databases on your homestead machine! So if your app relies on some data being available you will need to `ssh` into the server and run the migration & seeding scripts.

```bash
homestead destroy
homestead up
```

Perfect, you are all done. If everything went according to plan you should be able to see the default Laravel page when you view `http://myapp.dev` in your browser.
