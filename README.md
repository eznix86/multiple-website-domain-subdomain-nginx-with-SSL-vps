# Multiple website/subdomain using NGINX and let's encrypt
> This example has been tested with Docker and DigitalOcean VPS

This documentation is a how-to to make a VPS host multiple websites domain and subdomain using NGINX and docker.
 PR are allowed, and anyone who wish to update this documentation need to **fork** and **submit a PR**.

You will learn 
- How to create a NGINX reverse proxy
- Implement Let's encrypt for SSL certificate
- Using two distinct docker container to display on a domain and subdomain


# DNS Management

Before starting to setup your VPS, you need to manage your domain, and subdomain

## Create A records

On cloudflare, or DigitalOcean, create 2 two records of:
-  **type A**, having a hostname **any particular name**,  which directs to your **VPS IP**
> eg:
> subdomain.domain.com
> domain.com

|**TYPE**| **HOSTNAME**|**VALUE**|**TTL**| ...|
|---|---|---|---|---|
|  A | sudomain.domain.com | 188.177.11.132|  3600 |  ... |
|  A |  domain.com | 188.177.11.132  |  3600 |  ... |


##  Authoritative Nameservers

Now you've set your records, we need to manually set your nameservers. It depends on your domain provider. DigitalOcean gives a documentation on the matter, here is the [link](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars).

Once you've added your nameservers, you can check if the DNS propagation has been completed [here](https://www.whatsmydns.net). This will tell you if your IP and DNS are in sync.

# NGINX configuration (part 1)

Now that you've created your records, we can now start to manage our NGNIX stuffs. 

## Installation
-   Log into your  Server via SSH as the root user.
    ```
    ssh root@hostname-server
    ```
-   Use apt-get to update your Server.
    ```
    root@hostname-server:~# apt-get update
    ```
-   Install nginx.
    ```
    root@hostname-server:~# apt-get install nginx
    ```
   
-   Nginx may not start automatically, so you can to use the following command. Other valid options are "stop" and "restart".
```
 sudo /etc/init.d/nginx start
```
-   Check if all is okay by browsing at your domain name or IP address. You should see the default NGINX page.

## Configuration

We don't need NGINX page as web server here, we just need NGINX as a **reverse proxy**.

```
rm /etc/nginx/sites-enabled/default
```

Next we will add files to our ***conf.d*** folder.


# Docker servers

For this example, we will use 2 types of dockerized backend;
- A static website server
- A nodeJS server

## Configuration

First off, we need to install docker-compose to be able to run our docker-compose files.
```
apt install docker-compose
```

## Static Website 

### Structure of server
```bash
.
├── Dockerfile
├── docker-compose.yml
└── index.html
```

### Steps
```bash
cd ~
mkdir static-server
```
1. Create static `index.html` file
```bash
cat <<EOF >> index.html
<h1>Hello World</h1>
EOF
```
2. Create a `Dockerfile` file
```bash
cat <<EOF >> Dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EOF
```
3. Create a `docker-compose.yml` file
```bash
# this will create a docker, named static-web, exposed on port 8000
cat <<EOF >> docker-compose.yml
version: '2'

services:
  static-web:
    build: .
    ports:
     - "8000:80"
EOF
```
### Run a docker container
If you've got the structure right. Now type `docker-compose up -d` and  you can do a `docker ps` to see if the container is running.

> To stop the container, do `docker-compose stop`

## NodeJS Server with Docker Swarm

This didn't come from my personal knowledge, but it can be found on [this blog post](https://medium.com/@nirgn/load-balancing-applications-with-haproxy-and-docker-d719b7c5b231). For this nodeJS webserver will **use** this github README I've found [here](https://github.com/kametepe/docker-swarm-nodejs). 

### Structure of server
```bash
.
├── Dockerfile
├── docker-compose.yml
└── index.js
```
### Steps
> The code is found above.
> To stop the swarm you can do `docker swarm leave`, if it is the **leader**, (check [command here](https://docs.docker.com/engine/swarm/manage-nodes/).), you need to add `--force` flag.
> This will force the leader the leave the swarm and terminate the service.
---
##### Side Note:
> If one day, you need to do some docker **clean up** on your server, checkout this [link](https://gist.github.com/bastman/5b57ddb3c11942094f8d0a97d461b430).
> Or if you need to **erase everything**, use `docker system prune -a` if somehow you want to start over.

# NGINX configuration (part 2)

You've set up your containers ! We will now manage our NGINX to do a **domain** and **subdomain** for our server.


## Configuration

Now let's write our configuration files:
```bash
# let's get inside conf.d folder
cd /etc/ngnix/conf.d
```
#### Configuration for domain.com
```bash
# conf file for our domain.com
cat <<EOF >> domain.conf
server {
  listen 80;
  listen [::]:80;
  server_name domain.com;

  location / {
	proxy_pass http://static_server_ip/;
	proxy_buffering off;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP \$remote_addr;
  }
}
EOF
```

#### Configuration for subdomain.domain.com
```bash
# conf file for our subdomain.domain.com
cat <<EOF >> subdomain.domain.conf
server {
  listen 80;
  listen [::]:80;
  server_name subdomain.domain.com;

  location / {
	proxy_pass http://nodejs_server_ip/;
	proxy_buffering off;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP \$remote_addr;
  }
}
EOF
```
---
##### Side Note:
> Don't forget to replace the proxy `proxy_pass ` with your servers specific IP.

---
### Checking

Run `nginx -t` to check if everything is OK.
The result should be:
```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Run `ln -s /etc/ngnix/conf.d/domain.conf /etc/nginx/sites-enabled/` to enable to website.

And now, you can reload with `service nginx reload`.

Now go on your browser, point on your `domain.com` and your `subdomain.domain.com` and all should be fine.


##### Important Note:
> Don't forget to run the servers.
> First get inside each folder respectively,
> For static server is: `docker-compose up -d`

The static website container will run on port 8000

> For nodeJS in swarm mode: 
> ```bash
> docker build -t testimony .
> docker swarm init
> ```
- It may happen that prompt you to choose an IP.
- In this case, you can add  `--advertise-addr` flag following with the IP of your choice, but preferably use the **local IP** of your server.

> ```bash
> # Finally you run this:
> docker stack deploy --compose-file=docker-compose.yml production
> ```
The nodeJS container will run on port 85

# SSL on domain and subdomain

We will generate an SSL certificate for our domain and subdomain, for that we will use [Let's encrypt](https://letsencrypt.org/). It is a free SSL certificate provider. But the work is a bit complicated to set up, so we will use [Certbot](https://certbot.eff.org/) to leverage our work on this.

## Configuration

First off, we will find the version of our system, for my case, I'm using Ubuntu:
```bash
lsb_release -a
```
Result:
```bash
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.3 LTS
Release:        18.04
Codename:       bionic
```
Now navigate to [Certbot](https://certbot.eff.org/) website, and choose accordingly to the information you've got from finding your Operating System version, but don't forget to specify you are using **Nginx**.

Follow along, until you reach **step 4**: "Either get and install your certificates..." part, where you issue a certificate.

At this step, you will see:
```
certbot --nginx
```
Just follow along, and just fill in, then when it asks to redirect or no, select option 2 where it says **Redirect - Make all requests redirect to secure HTTPS access. **.

Now you are done !

##### Important Note:
> Go check your files in `/etc/nginx/conf.d/xxx.conf`
> You will notice that certbot automatically, and respectively added a configured SSL certificates for our domain  and subdomain.
> Note: It must be **regenerated** every 3 months.

Server Configuration

    /etc/nginx: The Nginx configuration directory. All of the Nginx configuration files reside here.
    /etc/nginx/nginx.conf: The main Nginx configuration file. This can be modified to make changes to the Nginx global configuration.
    /etc/nginx/sites-available/: The directory where per-site server blocks can be stored. Nginx will not use the configuration files found in this directory unless they are linked to the sites-enabled directory. Typically, all server block configuration is done in this directory, and then enabled by linking to the other directory.
    /etc/nginx/sites-enabled/: The directory where enabled per-site server blocks are stored. Typically, these are created by linking to configuration files found in the sites-available directory.
    /etc/nginx/snippets: This directory contains configuration fragments that can be included elsewhere in the Nginx configuration. Potentially repeatable configuration segments are good candidates for refactoring into snippets.

Server Logs

    /var/log/nginx/access.log: Every request to your web server is recorded in this log file unless Nginx is configured to do otherwise.
    /var/log/nginx/error.log: Any Nginx errors will be recorded in this log.

