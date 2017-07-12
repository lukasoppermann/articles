# Setting up let's encrypt & nginx on ubuntu 16.04 with docker

We will create an `nginx` proxy which in combination with the `certbot/cerbot` docker image will be used to create ssl certificates for our domains. Additionally the `nginx` server will be set up to server the certificates for the correct domains.

## Creating the server

For this article I am creating a [digital ocean](https://m.do.co/c/30eae8aadf01) server, which has a one-click install option for docker 17.05 on Ubuntu 16.04. If you want to try out digital ocean, sign up with this link: [https://m.do.co/c/30eae8aadf01](https://m.do.co/c/30eae8aadf01) to get a $10 credit.

While I add my ssh key via the GUI, you can easily [add your ssh keys manually](/secure-server-login-via-ssh).

Once the server and docker including **docker-compose** is set up wen can start.

## Understanding certbot

We will be using a docker container with `certbot/cerbot` to get our certificates. Certbot is the runs `webroot` the certification process for let's encrypt. What that means is that the certbot script will create a new folder in the *web root directory* (the root when accessing the server from the outside/internet) named `/.well-known/acme-challenge` within this folder a file with a hash as a name will be created. Afterwards a request is made to the let's encrypt service to try an access this file using the specified domain (the domain you want a certificate for). If this is successful a certificate and some additional files will be created.

For this process to work we need some kind of proxy e.g. **nginx** or apache. The proxy needs to define the web root and direct incoming requests into the correct directory.

## Setting up the proxy

### Docker-compose

## Certbot

### Creating certificates

### Renewing certificates

### Automating certificate renewal

### Convenicene scripts 