### note
EXPERIMENTAL - currently in development
## TODO
### stable-a01
- Letsencrypt
- docker run überarbeiten (parameter + startup.sh)
- Expire + GZIP (für assets)

### stable-a02
- Install Mailer
- add app.yml via run ?
- init dataservice
- Log(s!) Monitoring

## Init Instance
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y jq
```
## BUILD mswebdev:master
```bash
rm -r docker-cartodb
git clone -b master https://github.com/ms-webdev/docker-cartodb.git
docker build -t=mswebdev/cartodb docker-cartodb/
```
## RUN [!change Hostname]
change hostname
```bash
docker run -d -p 443:443 -e CARTO_HOSTNAME=cartodb-test.gopwa.de -e HTTPS=1 --name cartodb mswebdev/cartodb
```
## Docker CMDs
```bash
docker ps -a
docker logs cartodb

docker exec -it cartodb bash
docker stop cartodb
docker rm cartodb

docker system prune -a

docker inspect --format "{{json .State.Health }}" cartodb | jq
```
## Logs
```bash
nano /cartodb/log/production.log
nano /var/log/nginx/cartodb_error.log

nano /etc/redis/redis.conf
service redis-server restart
```

# --- END ---
## OpenSSL 
```bash
docker run -d -p 443:443 -e CARTO_HOSTNAME=<hostname> -e HTTPS=1 --name cartodb mswebdev/cartodb
```
## Letsencrypt
```bash
docker run -d -p 443:443 -p 80:80 -e CARTO_HOSTNAME=<hostname> -e HTTPS=1 -e LETSENCRYPT_EMAIL=<email adress> --name cartodb mswebdev/cartodb
```


docker-cartodb
==============

[![](https://images.microbadger.com/badges/image/sverhoeven/cartodb.svg)](https://microbadger.com/#/images/sverhoeven/cartodb "Get your own image badge on microbadger.com")
[![](https://images.microbadger.com/badges/version/sverhoeven/cartodb.svg)](https://hub.docker.com/r/sverhoeven/cartodb/)

This Docker container image provides a fully working cartodb development solution
without the installation hassle.

Just run the commands and then connect to http://cartodb.localhost with your browser.

The default login is dev/pass1234. You may want to change it when you run it for the outside.

It also creates an 'example' organization with owner login admin4example/pass1234.
Organization members can be created on http://cartodb.localhost/user/admin4example/organization

How to run the container:
-------------------------

```
docker run -d -p 80:80 -h cartodb.localhost sverhoeven/cartodb
```

The CartoDB instance has been configured with the hostname `cartodb.localhost`, this means the web browser and web server need to be able to resolve `cartodb.localhost` to an IP adress of the machine where the web server is running.
This can be done by adding cartodb.localhost alias to your hosts file. For example
```
sudo sh -c 'echo 127.0.1.1 cartodb.localhost >> /etc/hosts'
```
(For Windows it will be `C:\Windows\System32\drivers\etc\hosts`)

How to use a different hostname:
--------------------------------

For example to use `cartodb.example.com` as a hostname start with:
```
docker run -d -p 80:80 -h cartodb.example.com sverhoeven/cartodb
```

The chosen hostname should also resolve to an IP adress of the machine where the web server is running.

If you don't have a domain/subdomain pointing to your server yet, you can use the servers external ip address:
```
docker run -d -p 80:80 -h <servers-external-ip-address> sverhoeven/cartodb
```

Instead of setting hostname with `-h` you can also use the `CARTO_HOSTNAME` environment variable with:
```
docker run -d -p 80:80 -e CARTO_HOSTNAME=<hostname> sverhoeven/cartodb
```

HTTPS encryption
----------------

By default the Docker container runs unencrypted on port 80 and redirects to itself on port 80.

There are 2 ways to enable https encryption:

1. Use loadbalancer or reverse proxy to map https to http
2. Use embedded NGINX web server to perform encryption with automatic [Let's encrypt](https://letsencrypt.org/) certificate [deployment](https://certbot.eff.org/).

### 1. With load balancer or reverse proxy

Run container with
```bash
docker run -d -p 80:80 -e CARTO_HOSTNAME=<hostname> -e HTTPS=1 sverhoeven/cartodb
```

Configure load balancer or reverse proxy to accept traffic on https://<hostname>:443 and forward it to port 80 of the Docker container.

### 2. With automatic deployment

Run container with
```bash
docker run -d -p 443:443 -e CARTO_HOSTNAME=<hostname> -e HTTPS=1 -e LETSENCRYPT_EMAIL=<email adress> sverhoeven/cartodb
```

The `<email adress>` is used by Certbot as the account to register the domain at Let's Encrypt.

Let's encrypt has a [rate limit](https://letsencrypt.org/docs/rate-limits/) of a few generated certificates per domain per month, so you cannot just generate new certificates every time the container is restarted. So you should keep the generated certificates by mounting `/etc/letsencrypt`.

A cron job will try to renew the certificate each week.

Persistent data
---------------

To persist the PostgreSQL data, the PostGreSQL data dir (/var/lib/postgresql) must be persisted outside the Cartodb Docker container.

The PostgreSQL data dir is filled during the building of this Docker image and must be copied to the local filesystem and then the container must be started with the local copy volume mounted.

```bash
docker create --name cartodb_pgdata sverhoeven/cartodb
# Change to directory to save the Postgresql data dir (cartodb_pgdata) of the CartoDB image
docker cp cartodb_pgdata:/var/lib/postgresql $PWD/cartodb_pgdata
docker rm -f cartodb_pgdata
```

After this the CartoDB container will have a database that stays filled after restarts.
The CartoDB container can be started with
```
docker run -d -p 80:80 -h cartodb.example.com -v $PWD/cartodb_pgdata:/var/lib/postgresql sverhoeven/cartodb
```

Geocoder
--------

The external geocoders like heremaps, mapbox, mapzen or tomtom have dummy api keys and do not work.
No attempts have been made or will be made in this Docker image to get the external geocoders to work.

The internal geocoder is configured, but contains no data inside the image.

To fill the internal geocoder run
```
docker exec -ti <carto docker container id> bash -c /cartodb/script/fill_geocoder.sh
```

This will run the scripts described at https://github.com/CartoDB/data-services/tree/master/geocoder
It will use at least 5.7+7.8Gb of diskspace to download the dumps and import them.

How to build the image:
-----------------------

The image can be build with
```
git clone https://github.com/sverhoeven/docker-cartodb.git
docker build -t=sverhoeven/cartodb docker-cartodb/
```

The build uses the master branches of the [CartoDB GitHub repositories](https://github.com/CartoDB). A fresh build may fail when code requires newer dependencies then the Dockerfile provides or when code is not stable at the moment of building.
