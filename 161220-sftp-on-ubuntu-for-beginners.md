---
series: Servers for humans; 1
title: A beginners guide to setting up Ubuntu 16.04 and SFTP
tags: tag1, tag2
author: Lukas Oppermann
category: code
description: How to setup an Ubuntu 16.04 server and an sftp only user on an ubuntu server.
preview: Learn how to create an sftp user on ubuntu, who does not have ssh access and can only access a certain directory in her home directory.
---

> While front end and backend technology has evolved into a fairly user/developer friendly environment, interacting with servers still seems to be reserved for geeks and wizards. But with a little help, many things are actually quite doable.

In this post I am trying to guide you through the seemingly simple process of setting up some users with sftp access to their respective home folders, without allowing them normal ssh access through the console.

Personally I am using a [digitalocean.com](https://m.do.co/c/30eae8aadf01) server, they provide a pretty neat service for a reasonable price. If you want to sign up through <a href="https://m.do.co/c/30eae8aadf01" class="o-link o-link--decorated">this link</a> you will get a $10 credit. However any other server running Ubuntu 16.04 (or maybe even 14.04) should be fine as well.

## Basic setup

In case you just created a new server, I will walk you through adding your ssh key, disabling login via password and setting up your firewall.

If you have already done this, just skip ahead to the **Setting up SFTP** section.

Before we get going, note that I am working on a mac. If you are using a different OS, especially windows, some commands might differ. However, the general steps are the same.

### Login
Before we do anything, make sure you can login to your server using a password. With DigitalOcean you will get an email with the password for your server. You are required to change it the first time you log into your server. So log in, and follow the prompts to change your password. Once you successfully changed your password, you can logout using the `exit` command.

### SSH Key
To be able to login without a password we need an ssh key pair which we will copy to our server.

If you already have a pair of keys, you can use this key instead of creating a new one. To view all key pairs, use this command:

```bash
$ ls ~/.ssh/
```

If you see `id_rsa` and `id_rsa.pub`, you are all set and can use those keys, the benefit of using the `id_rsa` key is, that you will be able to login without specifying the key name.

To create a new public key pair use the `ssh-keygen` command, if you do not have any ssh key yet, use the default name `id_rsa` instead of `devserver_rsa`.

```bash
$ ssh-keygen -t rsa
Generating public/private rsa key pair.

# take care to not overwrite a key you are currently using
# choose a new name that describes the keys usage
Enter file in which to save the key (/Users/YOURUSERNAME/.ssh/id_rsa): /Users/YOURUSERNAME/.ssh/devserver_rsa

# you can choose a strong password or leave it blank
# if you enter a passphrase you will need to enter it when you log into your server
Enter passphrase (empty for no passphrase):
Enter same passphrase again:

Your identification has been saved in  /Users/YOURUSERNAME/.ssh/devserver_rsa.
Your public key has been saved in /Users/YOURUSERNAME/.ssh/devserver_rsa.
The key fingerprint is:
SHA256:ptdHTSJHm8aFXjevLQy3TLCHARgstNS8wQml7M6//lI yourname@something
The key's randomart image is:
+---[RSA 2048]----+
|     .+Bo+.....  |
|     o.oO  o++...|
|      +. o..B*..o|
|     .  .  +=++ .|
|      . S   .B.+ |
|     o o .E.  * .|
|      + ... .  . |
|       o.  .     |
|       .++.      |
+----[SHA256]-----+
```

Now you need to add the key to your server. DigitOcean offers a convenient GUI when creating a server, but you can easily do this in the terminal as well. Just make sure to reference the correct `.pub` file and replace`123.45.56.78` with your servers ip.

```bash
$ cat ~/.ssh/devserver_rsa.pub | ssh root@123.45.56.78 "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
```

If you just want to print the key, to copy it to, just use the `cat` command.

```bash
$ cat ~/.ssh/devserver_rsa.pub
ssh-rsa AAAAxchuMZERJsWrbRkzwsDeBRgbgFWfkeaEDxUjRt9pybmjXUDXdaMYuGHf+FzLukXJsIKzPmsWrbRkzwsDeBRgbg+v/  
EDxUjR/mjXUDXdaMY+kzwsDeBRgbgFWfkeaEDxUjRt9pybmjXUDXfkeaEDxUjRt9pybmjXUDXdaMYuGHf/
MZERJsWrbRkzwsDeBRgbgFWfkeaEDxUjRt/rbRkzwsDeBRgbg/xchuMZERJsWrbRkzwsDeBRgbgFWfkeaEDxUjRt9
pybmjXUDXdaMYuGHfxchuMZERJsWrbRkzwsDeBRgbgFWfkeaEDxUjRt9pybmjXUDXdaMYuGHfWpn   
lukasoppermann@Lukass-MacBook-Pro.local
```

### Test SSH login without password
Before we go ahead and disable the login via password we better make sure we can actually login using the SSH key. Otherwise we might permanently lock ourselves out of our own server.

1. If you are connected to your server, disconnect by typing `exit` and confirm with the `return` key.
2. Reconnect using the `ssh` command, replacing the IP with your servers IP.

```bash
$ ssh root@123.45.56.78
```

If you have used an SSH key other than `id_rsa` you will need to use the following command instead, replacing the IP and `devserver_rsa` with the name of your key.

```bash
$ ssh -i /Users/$(whoami)/.ssh/devserver_rsa.pub root@123.45.56.78
```

3. You will be asked if you want to connect to the server. Confirm by typing `yes` and hitting return.

```bash
The authenticity of host '123.45.56.78 (123.45.56.78)' can't be established.
RSA key fingerprint is b1:4d:37:57:ca:35:4d:5f:f3:59:cd:c0:c4:48:86:11.
Are you sure you want to continue connecting (yes/no)? yes
```


4. If everything went according to plan you should now be connected.

**Tip:** If you are using a different key than `id_rsa` you can create an `alias` to make connecting easier.

```bash
# create an alias to connect with the command ssh-devserver
$ alias ssh-devserver="ssh -i /Users/$(whoami)/.ssh/devserver_rsa.pub root@123.45.56.78"
```

Open a new terminal window or reload your profile to have this alias available.

### Disable SSH login via password
Now that we know we can login with the SSH key we can disable SSH access via password.

>> This is recommended because it prevents attackers from brute forcing their way into your server, by trying random passwords.

Connect to your server and open the `sshd_config` file.

```bash
$ ssh root@123.45.56.78
$ sudo nano /etc/ssh/sshd_config
```

Change `PermitRootLogin` in the `sshd_config` file to `without-password` to restrict login via SSH key.

```bash
PermitRootLogin without-password
```

Close and save the `sshd_config` file by pressing `ctrl +  x`, type `y` and confirm with the `return` key. Now you just need to reload the ssh service.

```bash
$ service ssh reload
```

To follow best practices it is advised that you now create a new `sudo` user to use instead of the root, as root as near unlimited power and you can easily mess up your server. For brevities sake I will leave this for another article.

### Setting up Ubuntus firewall UFW
First we need to allow SSH access to not lock ourselves out of our own server. Once this is allowed, we can enable the firewall and check the status.

```bash
# allow ssh access through firewall
$ sudo ufw allow ssh

# enable firewall
$ ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

# check firewall status
$ ufw status
To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
```

**DO NOT** `exit` your server with the firewall active if the two entries above are not shown. If you can't get it to work, you can disable the firewall again using the `disable` command. Check the status afterwards to make sure the firewall is  actually disabled.

```bash
$ ufw disable
Firewall stopped and disabled on system startup

$ ufw status
Status: inactive
```

If you are planning on using your server to host any kind of application, you will probably want to allow `http` and `https`. In case you need to delete a rule, the easiest way is to used the numbered status, this way you can just use the `ufw delete` command with the number of the rule you want to delete.

```bash
# allow http & https
$ ufw allow http
$ ufw allow https

# show the numbered firewall status
$ ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22                         ALLOW IN    Anywhere
[ 2] 80                         ALLOW IN    Anywhere
[ 3] 443                        ALLOW IN    Anywhere
[ 4] 22 (v6)                    ALLOW IN    Anywhere (v6)
[ 5] 80 (v6)                    ALLOW IN    Anywhere (v6)
[ 6] 443 (v6)                   ALLOW IN    Anywhere (v6)

# delete a rule, e.g. 6: 443 (v6)
$ ufw delete 6
```

## Setting up SFTP

First we create a new user group `sftpuser` which we will use to restrict the directories accessible through sftp. Afterwards we create our first user `lukas` and assign this user to the `sftpuser` group. The `--ingroup` flag assigns the user to the desired group and additionally prevents the creation of the default group `lukas` (the username). You will be prompted to enter a password for the new user. Afterwards you may fill in some optional information about your user.

```bash
$ addgroup sftpuser
Adding group 'sftpuser' (GID 1000) ...
Done.

$ adduser lukas --ingroup sftpuser
Adding user 'lukas' ...
Adding new user 'lukas' (1000) with group 'sftpuser' ...
Creating home directory '/home/lukas' ...
Copying files from '/etc/skel' ...
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
Changing the user information for lukas
Enter the new value, or press ENTER for the default
	Full Name []:
	Room Number []:
	Work Phone []:
	Home Phone []:
	Other []:
Is the information correct? [Y/n] Y
```

By default Ubuntu will create a directory with the users name in the `/home` directory. However we only want the user to be able to access a `files` directory within an `ftp` directory. For this we need to create the `ftp` and `files` directory, change the ownership of the users home directory (`lukas`) and `ftp` to `root:root` and change the access rights for the `files` directory to our user `lukas:sftpuser`.

```bash
$ mkdir /home/lukas/ftp
$ mkdir /home/lukas/ftp/files
$ chown root:root /home/lukas
$ chown root:root /home/lukas/ftp
$ chown lukas:sftpuser /home/lukas/ftp/files

# if necessary make the folder writable
$ chmod ug+rwX /home/lukas/ftp/files

# check for access rights
$ namei -l /home/lukas/ftp/files
drwxr-xr-x root root /
drwxr-xr-x root root home
drwxr-xr-x root root lukas
drwxr-xr-x root root ftp
drwxr-xr-x lukas sftpuser files
```

Now we need to set the home directory for our user to `/home/lukas/ftp`. You can check the home directory of another user with the `eval` command.

```bash
$ usermod -d /home/lukas/ftp lukas

# to check a users home directory
$ eval echo "~lukas"
/home/lukas/ftp
```

The last thing we need to do is to modify the `/etc/ssh/sshd_config` file. First we need to comment out the old `Subsystem` line and add a new one `Subsystem sftp internal-sftp` instead. This newer version is needed for the `ChrootDirectory` config we will add at the end.

```bash
$ nano /etc/ssh/sshd_config

# Within  v:
# Search for the line below, add the # to disable it and add the new Subsystem line

#Subsystem sftp /usr/lib/openssh/sftp-server
Subsystem sftp internal-sftp
```

Afterwards add the code below to the end of the `at very end of the file` file. The `Match` will only activate this for users in the `sftpuser` group and allow the access via sftp but only to their home directory `ChrootDirectory %h`. Make sure the lines after the `Match Group sftpuser` are indented to only apply those the the match.

```bash
Match Group sftpuser
    ChrootDirectory %h
    X11Forwarding no
    AllowTcpForwarding no
    ForceCommand internal-sftp
```

Now we just need to restart the SSH service and you should be able to connect with your new user either via an ftp tool, or the `sftp` command. SSH access should be disabled, so the new user is an sftp only users with no `shell` and no `sudo`.

```bash
$ service ssh restart

# check if sftp access is working
$ sftp lukas@123.45.56.78
lukas@123.45.56.78`s password:
Connected to 123.45.56.78.
sftp> bye

# check if ssh access is disabled
ssh lukas@123.45.56.78
lukas@123.45.56.78's password:
This service allows sftp connections only.
Connection to 123.45.56.78 closed.
```
