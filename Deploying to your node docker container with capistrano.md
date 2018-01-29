Creating a node container on docker and deploying to it via capistrano
========

# Adding a deploy user

**How to do with cloud-init…**
https://cloudinit.readthedocs.io/en/latest/topics/examples.html

Ideally you will want to disable the `root` user login and deploy with a dedicated user. So lets create a user named `deployment`.

```
adduser deployment --ingroup sudo --gecos ""
```

Now we need to also add this new user to the `docker` group, so it can use docker without sudo later on.

```
adduser deployment docker
```

# CloudInit
 Add users to the system. Users are added after groups are added.
 
```yams
users:
  - default
  - name: deploy
    gecos:
    primary-group: sudo
    groups: docker
    ssh-import-id: foobar
    lock_passwd: false
    passwd: $6$j212wezy$7H/1LT4f9/N3wpgNunhsIqtMj62OKiS3nyNwuizouQc3u7MbYCarYeAHWYPYb2FT.lbioDm2RrkJPb9BZMN1O/
  - name: barfoo
    gecos: Bar B. Foo
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    ssh-import-id: None
    lock_passwd: true
    ssh-authorized-keys:
      - <ssh pub key 1>
      - <ssh pub key 2>
```