# Setting up let`s encrypt & nginx on ubuntu 16.04 with docker

We will create an `nginx` proxy which in combination with the `certbot/cerbot` docker image will be used to create ssl certificates for our domains. Additionally the `nginx` server will be set up to server the certificates for the correct domains.

## Creating the server

For this article I am creating a [digital ocean](https://m.do.co/c/30eae8aadf01) server, which has a one-click install option for docker 17.05 on Ubuntu 16.04. If you want to try out digital ocean, sign up with this link: [https://m.do.co/c/30eae8aadf01](https://m.do.co/c/30eae8aadf01) to get a $10 credit.

While I add my ssh key via the GUI, you can easily [add your ssh keys manually](/secure-server-login-via-ssh).

Once the server and docker including **docker-compose** is set up we can start.

The files we create will need to be placed within a directory on your server, e.g. your users home directory.

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

Now that this file exists you can run `docker-compose -d` and it should run without an error. 

## Certbot

Now that we have our nginx server ready it is time to create our certificates. Before you can do this you will need to point a domain to your server, I created `staging.vea.re`. Make sure to substitute it for your domain.

### Creating certificates

Using docker we can `run` a container with the `--rm` flag, which means it will be removed once it has served its purpose. This is ideal for our `certbot/certbot` container.

```bash
docker run -it --rm \
	--name letsencrypt \
  -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
  --volumes-from nginx \
  certbot/certbot certonly \
  --webroot \
  --webroot-path /var/www/html \
  --agree-tos \
  --staging \
  --dry-run \
  -m your@email.com \
  -d staging.vea.re
  
```

I will go through all flags one-by-one, make sure to remove the `--dry-run` flag once you want to actually create a certificate and the `--staging` flag if you want a real one.

- `-it` makes the container interactive which we want because we may need to interact with questions from `verbot`

- `--rm` removes the container once it exits

CAN WE DELETE THEM?=??
- `--name` gives the container a name
- `-v` mounts volumes, we

- `--volumes-from` mounts all volumes that are mounted to a specific container `nginx` in our case. We want this because certbot must place our certificates here.

Now we run `certbot/certbot certonly`, `certbot/certbot` specifies the container on hub.docker.com. Since we don't want certbot to create a server but use our nginx server instead we tell it to only create the certificates using the `certonly` command. The following flags are for certbot now.

- `--webroot` tells certbot to run in the webfoot certification mode

- `--webroot-path` tells it which path to use as the web root, `/var/www/html` for us

- `--agree-tos` automatically agrees to the terms of services, instead of asking us, this is important for automating it

- `--staging` creates a staging certificate. Be careful with non-staging certificates during testing, as there is a limited number of requests you are allowed to make. *(Remove the flag to create a production certificate)*

- `--dry-run` will test everything but not actually create a certificate. Make sure this runs without an error before creating a certificate. *(Remove the flag to create a certificate)*

- `-m` is used to tell certbot which email address to use for registration

- `-d` lets you specify which domain you want to create a certificate for

### Renewing certificates

At some point you will need to renew your certificates. They are only valid for 90 days after all.

For this we can use certbot as well and run the `renew` command with it.

```bash
docker run -it --rm \
  --volumes-from nginx \
  certbot/certbot renew
```

### Testing your certificate

To make sure your certificate is working you can add the following at the end of your `/nginx/conf.d/default.conf`. 

```
server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	server_name staging.vea.re; # replace with your domain
	
	# replace your domain in both paths
	ssl_certificate /etc/letsencrypt/live/staging.vea.re/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/staging.vea.re/privkey.pem;

	location / {
		root         /var/www/html;
	}
}
```

You will need to make nginx reload the config file before this change will take effect. The easiest way to do this, is to run the following command.

```bash
# it should tell you that your file is fine
docker exec -it nginx nginx -t

# this will reload your file
docker exec -it nginx nginx -s reload
```

Now if you place `index.html` with `<h1>Hello world!</h1>` into the `/var/www/html` directory, you should be able to reach it via the browser at `https://your-domain.com/index.html`.

### Additional security measures

You need to make sure that your server, proxy and certificates are secure. The first thing to do is to create strong Diffie–Hellman parameters.

#### Diffie–Hellman Parameters
The Diffie–Hellman key exchange is a method of exchanging cryptographic keys securely. The Diffie–Hellman parameters are basically used to secure this exchange, so you want strong ones.

Creating them is simple, on your server run the following command:

```bash
# first move into your letsencrypt directory
cd ~/letsencrypt

# now create your parameters
sudo openssl dhparam -out dhparam.pem 2048
```

Now you just need to make sure it is actually used. 
This can be done by placing one line into your `/nginx/conf.d/default.conf` file within the `server` block:

```
ssl_dhparam /etc/letsencrypt/dhparam.pem;
```

Since we mapped then `~/letsencrypt` to the nginx docker container into `/etc/letsencrypt`, you should have the file available. Make sure to test (`nginx -t`) & reload (`nginx -s reload`) your nginx server, like we did before.

#### Other security settings
There are some other security settings & ssl parameters you will want to add. To make it easier to include them into every server block I extracted them into a new file named `ssl-params.conf` in the new directory `/nginx/includes`. I also moved the `ssl_dhparam` line here.

```bash
# from https://cipherli.st/
# and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_stapling off;

add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";

# load the diffie–hellman parameter
ssl_dhparam /etc/letsencrypt/dhparam.pem;
```

*A fair warning, I am by no means an expert on nginx security, so you might want to read up in it somewhere else as well.*

Now this file needs to be included within your server block instead of the `ssl_dhparam` line:

```bash
include /etc/nginx/includes/ssl-params.conf;
```

### Automating certificate renewal
Wouldn't it be ideal if you would not have to worry about your certificates at all? Well, this can be achieved by making certbot renew your certificates automatically, via a simple `cron job`.

For this I suggest creating a new directory `letsencrypt_scripts` in which we can create a file named `cron_letsencrypt`:

```bash
#!/bin/sh

# the directory to store log files
log_dir=../logs/nginx/letsencrypt/

# run the renew script an store the result in $output
output=$(docker run -it --rm \
  --volumes-from nginx \
  certbot/certbot renew \
  --text 2>&1)
  
# Get Date
date=`date +%Y-%m-%d_%H%M%S`

# create log folder if it does not exist
mkdir -p $log_dir

# Write output to log file
cat > ${log_dir}${date}_letsencrypt.txt << EOF
Date: $date
$output
EOF

# send email if mail command is available
# replace server@domain.com & your@email.com
if type "mail" &> /dev/null; then
  echo "$output" | mail -s "Daily SSL Revalidation update from ${date}" -A ${log_dir}${date}_letsencrypt.txt -r 
  server@domain.com your@email.com
fi

# remove log files older than 40 days
find $log_dir -type f -mtime +40 -delete
```

The script runs the command, logs the output to a file within `/logs/nginx/letsencrypt/` and sends an email to you with the log as long as `mailutils` is installed.

If you want to install it, you can run the following lines ion ubuntu:

```bash
apt-get update
apt-get install mailutils
```

Now all that is left to do is to add a line to the `crontab`. 

```bash
crontab -e
```

Now you can add a line at the very bottom, like:

```bash
30 18 * * *  /home/letsencrypt_scripts/cron_letsencrypt
```

The cron time format is shown below. You can just [google crontab time format](https://www.google.com/search?q=crontab+time+format) to get a better idea of how it works.

```bash
* * * * * *
| | | | | | 
| | | | | +-- Year              (range: 1900-3000)
| | | | +---- Day of the Week   (range: 1-7, 1 standing for Monday)
| | | +------ Month of the Year (range: 1-12)
| | +-------- Day of the Month  (range: 1-31)
| +---------- Hour              (range: 0-23)
+------------ Minute            (range: 0-59)
```

Once you are done save your file and you should be all set. You probably should run the task every day so that you have  ample time to deal with any renewal issue if they arrive. 