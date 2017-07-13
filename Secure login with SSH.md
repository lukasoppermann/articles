# Secure Server login via SSH with an ssh-key

> Disabling password login and disallowing login with the root user mitigates the risk of attackers compromising your server.

This article starts with the assumption that you have a clean installation of Ubuntu on your server. I am working on 16.04 but the steps should be similar for other versions. For other Linux distributions you might need to research the correct commands.

## First login
If you did not add an ssh key via the GUI when creating the server, you will have either created a root password, or you will have gotten it via other means, like email with digital ocean. (Note that digital ocean will not send a password and automatically disabled password logins if you add an ssh key via the GUI.)

To log into your server use the `ssh` command replacing `123.45.56.78` with your servers ip. (Passwords are never visible in the commandline, so just type away.)

```bash
ssh root@123.45.56.78

The authenticity of host '123.45.56.78 (123.45.56.78)' can′t be established.
ECDSA key fingerprint is SHA256:mX1fCAc5cyf2bG7BZnBPhrrmIKANdBtWzk676MgqhSs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '123.45.56.78' (ECDSA) to the list of known hosts.

root@123.45.56.78's password:
You are required to change your password immediately (root enforced)
```

You will be asked to add the server to your list of known hosts, confirm this by typing `yes` and hitting return.

Afterwards you must enter your `root` user password. You will mist likely be asked to change you password before continuing (at least if you got a random password via email).

Normally you will first need to enter your current password and afterwards enter a new password as well as confirm it by entering it again. 

```bash
# neither your password nor any * characters will be shown
Changing password for root.
(current) UNIX password:
Enter new UNIX password:
Retype new UNIX password:
```


## Creating a sudo user

We still want to be able to log in and work on the server, so we need to create a new user with `sudo` privileges.

```bash
$ adduser lukas --ingroup sudo --gecos ""
```

My user is named `lukas`, the `--ingroup` flag allows us to assign the user to one group (if you leave it out a group with the users name will be created). The `--gecos` flag allows us to assign details like a phone number. Since this is not interesting, we just provide an empty string `""` to avoid answering all the questions.

You will be asked to enter a password and confirm it. Make sure it is a **secure** password and that you remember it. It will be needed to use the `sudo` command.

Before we move on, make sure it all worked.

```bash
# switch to your new user
$ su lukas

# run any command with sudo
$ sudo -v
[sudo] password for lukas:

# If you do not have sudo rights you will see this message
Sorry, user lukas may not run sudo on veare.localdomain.
```

To **disconnect** from the server simply type `exit` and hit return.

## Allow access via SSH

If you are using services like github you probably already have an ssh key on your computer. To see if and which keys you have, run the following command:

```bash
# list everything in the ~/.ssh/ directory
$ ls ~/.ssh/
authorized_keys  config   id_rsa  id_rsa.pub   known_hosts
```

The default private key is in `id_rsa` and the public key in `id_rsa.pub`. I recommend using them because you will not have to specify which key to use when logging in.

### Creating a new key pair

If you have no key pair yet, or want to create a new key pair you can use the `ssh-keygen` command. You will be asked where to store it. In the example below I am creating a pair named `devserver_rsa`. If you have no key pair yet, just hit return and stick to the default. 

However if you do already have one, **do not** overwrite your old pair. Choose a different name for your new pair.

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

### Transferring the key pair

We must copy the public key pair to the home directory of both our `root` and our sudo user (`lukas` for me) on our server, to allow them to log using ssh. Make sure to replace `123.45.56.78` with your servers ip.

In the examples below I am using the default public key, if you create one with a different name, replace `id_rsa.pub` with your name, e.g. `devserver_rsa.pub`.

```bash
# for the root user
$ cat ~/.ssh/id_rsa.pub | ssh root@123.45.56.78 "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"

# for the lukas user
$ cat ~/.ssh/id_rsa.pub | ssh lukas@123.45.56.78 "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
```

I connect with both user simultaneously so that `~` refers to each users home directory. Alternatively you could just use root and copy the other users ssh key into `/home/lukas`.

### Verifying the SSH login via ssh-key

You should now be able to login without using a password. To do so, simply run the `ssh` command again. Try this for both users.

```bash
# login with the root user
$ ssh root@123.45.56.78
# disconnect
$ exit
# login with the lukas user
$ ssh lukas@123.45.56.78
# disconnect
$ exit
```

## Disabling Root-Login and login via password

Once you verified that login via ssh-key (without password) works, and you can use `sudo` with your new user, you should disable login using passwords as well as the `root` user login. 

This is a security precaution. The ssh key is a secret that needs to be stolen from your computer while a password can be brute-forced or socially, which makes it less secure. Disabling it means only users with your ssh key can log into the server.

The `root` user has god-like powers on your server, which can pose quite a security risk as well as making it easy for you to break something on accident. Once logged in a `root` user can run sudo-like commands without a password, while a sudo user needs to enter the password (kind of 2-factor).

Disabling the password is rather simple. Login to your server and edit `/etc/ssh/sshd_config`.

```bash
$ sudo nano /etc/ssh/sshd_config
```

Within the file look for the `PasswordAuthentication` line and change its value to `no`.

```
PasswordAuthentication no
```

Before you close this file do the same for `PermitRootLogin` to disable `root` login.

```
PermitRootLogin yes
```

Once changed you can exit and save by pressing `ctrl + x`, pressing `y` to save the changes and afterwards `return` to confirm overwriting the current file. 

To make those changes take effect you need to reload the `ssh` service.

```bash
$ sudo systemctl reload sshd
```

To confirm it worked, disconnect from your server by typing `exit` and try to connect as the `root` user.

```bash
$ ssh root@123.45.56.78
Permission denied (publickey).
```

You should not be able to connect. However your sudo user should be able to log into the server just like before. 

And this is it. There are many other aspects to a secure server, for example firewalls, but one very important step is done now. Pad yourself on the back.

