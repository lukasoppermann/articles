# Setting up let`s encrypt & nginx on ubuntu 16.04 with docker

We will create an `nginx` proxy which in combination with the `certbot/cerbot` docker image will be used to create ssl certificates for our domains. Additionally the `nginx` server will be set up to server the certificates for the correct domains.

## Creating the server

For this article I am creating a [digital ocean](https://m.do.co/c/30eae8aadf01) server, which has a one-click install option for docker 17.05 on Ubuntu 16.04. If you want to try out digital ocean, sign up with this link: [https://m.do.co/c/30eae8aadf01](https://m.do.co/c/30eae8aadf01) to get a $10 credit.

While I add my ssh key via the GUI, you can easily [add your ssh keys manually](/secure-server-login-via-ssh).

Once the server and docker including **docker-compose** is set up wen can start.

## Understanding certbot

We will be using a docker container with `certbot/cerbot` to get our certificates. Certbot is the runs `webroot` the certification process for let`s encrypt. What that means is that the certbot script will create a new folder in the *web root directory* (the root when accessing the server from the outside/internet) named `/.well-known/acme-challenge` within this folder a file with a hash as a name will be created. Afterwards a request is made to the let`s encrypt service to try an access this file using the specified domain (the domain you want a certificate for). If this is successful a certificate and some additional files will be created.

## Setting up the proxy

For the above process to work we need some kind of web server/proxy e.g. **nginx** (which we will use) or apache. The proxy needs to define the web root and direct incoming requests into the correct directory. 

### Docker-compose
You will need to destroy and re-create your docker services at some point in the future. You do this a lot during development but also if the server fails and needs to be restarted or if you do blue-green deployment, etc. you will need an easy and fast way to bring your docker services up without a headache. Luckily docker has just such a solution: **docker-compose**. 

With docker-compose you can specify your services, networks and volumes in a YAML file and initialise everything with a simple `docker-compose up` command.

Our `docker-compose.yml` file will have 3 main sections:

1. `version` specifies the docker-compose version are using
2. `services` describes our containers
3. `networks` are used for internal communication

```yaml
# docker-compose version
version: '3'

# our containers
services:

# used for communication within docker
networks:
```

#### Networks

We simply need one network, which I named `appnet`. All containers within the same network can talk to each other.

**Note:** Make sure to use correct indentation and cases, as YAML is indentation sensitive (only spaces are allow, no tabs!) and case sensitive, you can [dig into YAML here](https://learn.getgrav.org/advanced/yaml) if you want.

```yaml
# used for communication within docker
networks:
    appnet:
        driver: "bridge"
```

#### Services

```yaml
# our containers
services:
    # we only have on service we name nginx
    nginx:
        # it is build from the nginx:alpine image
        image: nginx:alpine # using alpine for newer openssl for http2 with ALPAN
        
        container_name: nginx
        
        # these volumes are mounted to the nginx container
        volumes:
            
            # mount the folder ../letsencrypt from our server to the folder /etc/letsencrypt on the container
            - ../letsencrypt/:/etc/letsencrypt
            
            # mount ./nginx/var/www/html to the /var/www/html on the container
            - ./nginx/var/www/html:/var/www/html
            
            # mount ./logs/nginx/ to the /var/log/nginx on the container, rw = read-write
            - ./logs/nginx/:/var/log/nginx:rw
            
            # mount ./nginx/includes to the /etc/nginx/includes on the container, ro = read-only
            - ./nginx/includes:/etc/nginx/includes:ro
            
            # mount ./nginx/conf.d/default.conf to the /etc/nginx/conf.d/default.conf on the container
            - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf
            
            # mount ./sites to the /sites on the container (will hold our node app)
            - ../sites:/sites
        
        # ports below are reachable from the outside
        ports:
            # we need to open port 80 for the verification process before we have an ssl certificate
            - "80:80"
            # 443 is the ssl port so we need to expose it
            - "443:443"
        
        # makes container restart on server restart
        restart: always
        
        # used for interal communication within docker
        networks:
            # list all networks we want to connect to
            - appnet
```

Our entire file now looks like this:

```yaml
version: '3'
services:
    nginx:
        # using alpine for newer openssl for http2 with ALPAN
        image: nginx:alpine

        container_name: nginx

        # these volumes are mounted to the nginx container
        volumes:
            # mount the folder ../letsencrypt from our server to the folder /etc/letsencrypt on the container
            - ../letsencrypt/:/etc/letsencrypt
            # mount ./nginx/var/www/html to the /var/www/html on the container
            - ./nginx/var/www/html:/var/www/html
            # mount ./logs/nginx/ to the /var/log/nginx on the container, rw = read-write
            - ./logs/nginx/:/var/log/nginx:rw
            # mount ./nginx/includes to the /etc/nginx/includes on the container, ro = read-only
            - ./nginx/includes:/etc/nginx/includes:ro
            # mount ./nginx/conf.d/default.conf to the /etc/nginx/conf.d/default.conf on the container
            - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf
            # mount ./sites to the /sites on the container (will hold our node app)
            - ../sites:/sites
        ports:
            - "80:80"
            - "443:443"
        # makes container restart on server restart
        restart: always
        networks:
            - appnet
# used for interal communication within docker
networks:
    appnet:
        driver: "bridge"
```


### Starting your container
To start your services you need to run `docker-compose up -d`. The `-d` flag starts it in demon mode, so you are not stuck listening to the container. If you forget the `-d` you can exit by pressing `ctrl` + `c` this will however kill the services. 

When starting the services now you will get an error:

```
Are you trying to mount a directory onto a file (or vice-versa)? Check if the specified host path exists and is the expected type
```

This is because `docker-compose` automatically creates the folders we specify in the volume section of our service, in case they don't exist. However nginx expects `/conf.d/default.conf` to be a file and not a folder, so we need to create it first.

In the folder where your `docker-compose.yml` lives create `/nginx/conf.d/default.conf`:

```
# don't redirect proxy
proxy_redirect  off;
# turn off global logging
access_log off;
# logging format
log_format compression '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $bytes_sent '
                       '"$http_referer" "$http_user_agent" "$gzip_ratio"';
############################
#
# allow let`s Encrypt on port 80 for all domains
#
server {
  listen 80;
  listen [::]:80;
  server_name ~. ;

  location /.well-known/acme-challenge {
    root /var/www/html;
    default_type text/plain;
  }

  location / {
    return 301 https://$host$uri;
  }
}
```

After defining some basic setting for redirection and logging we move on to the `server` directive. In here we only listen on port `80`, for non-SSL connections. The only connection that we allow is to anything within the `/.well-known/acme-challenge` directory.

For requests to this directory the root is `/var/www/html`, which we mounted into our service, so we will be able to put our websites there from the outside.

`default_type text/plain` sets the default content-type to `text/plain` which we need for let`s encrypt. 

The last part redirects all requests on port `80` that are to anything outside the `/.well-known/acme-challenge` directory to `https`.



## Certbot

### Creating certificates

### Renewing certificates

### Testing your certificate
server {
server_name staging.lukasoppermann.de;
    ssl_certificate /etc/letsencrypt/live/staging.lukasoppermann.de/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/staging.lukasoppermann.de/privkey.pem;
listen 443 ssl http2;
listen [::]:443 ssl http2;
location / {
        root         /var/www/html;
}
}

### Automating certificate renewal

### Convenicene scripts 