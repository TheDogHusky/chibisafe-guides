---
title: Running with Docker and Nginx Proxy Manager
summary: This guide shows you how to run chibisafe with Nginx Proxy Manager (or NPM) through Docker without the internal Caddy server.
---

# Running chibisafe with Nginx Proxy Manager through Docker
> This guide shows you how to run chibisafe with Nginx Proxy Manager (or NPM) through Docker without the internal Caddy server.

In order to install chibisafe and run it with Nginx Proxy Manager there are a few things you need to set up on your system beforehand. This guide assumes you are using Ubuntu/Debian so feel free to adjust commands as you see fit based on your distro.

:::tip
  This guide assumes you are an advanced user. If you are not familiar with Docker, Nginx Proxy Manager and development in general, please install chibisafe normally through docker and follow the [official guide](/docs/installation/running-with-docker).
:::

### Pre-requisites

Before running chibisafe make sure you install docker and docker-compose. Please refer to the [official Docker documentation](https://docs.docker.com/get-docker/) for installation instructions.

You will need to set up Nginx Proxy Manager as well, on port 80, 443 and 81. Go to the [official Nginx Proxy Manager documentation](https://nginxproxymanager.com/guide/#quick-setup) for installation instructions.

You will also need a FQDN (Fully Qualified Domain Name) pointing to your server. You can use a service like [Cloudflare](https://www.cloudflare.com/) to get a domain.

### Installing

This guide assumes you will use Docker to run chibisafe. If you want to run it manually, just follow the [manual guide](/docs/installation/running-manually) and install Nginx Proxy Manager like we said.

First, move to a place where you can install it:

```bash
cd /home/<your user>
```

:::tip
  As Docker *by default* does not run as your user, this *may not* be the best location to put chibisafe in.
  The more "appropriate" location would be on `/srv`, so change the place where chibisafe is installed accordingly if you want to follow best practices.
:::

Then, make a directory for chibisafe and move to it:

```bash
mkdir chibisafe
cd chibisafe
```

Now, you need to create a docker-compose.yml file. You can use the following template:

```yaml
version: "3.7"

services:
  chibisafe:
    image: chibisafe/chibisafe:latest
    environment:
      - BASE_API_URL=http://chibisafe_server:8000
    expose:
      - 8001
    restart: unless-stopped
    ports:
      - 8001:8001

  chibisafe_server:
    image: chibisafe/chibisafe-server:latest
    volumes:
      - ./database:/app/database:rw
      - ./uploads:/app/uploads:rw
      - ./logs:/app/logs:rw
    expose:
      - 8000
    restart: unless-stopped
    ports:
      - 8000:8000
```

> What did we change?
> We removed the caddy section, as we won't need it. We also exposed to the host the ports 8000 and 8001, so Nginx Proxy Manager can access them.

:::tip
  In these changes, we removed the static file serving. You will need to serve your chibisafe files from a different location with either a web server or a Docker container image that allows this, and then configure chibisafe accordingly.
  Head over to the [file serving](#serve-the-uploads) section for more information on how to do this.
:::

Now, you can run `docker-compose up -d` to start chibisafe.

:::tip
  This way of hosting chibisafe will NOT allow you to customize the ports which chibisafe listens on. So if anything (like portainer) uses the `8000` and `8001` port, you'll get an error and will need to change the port the other thing listening on 8000 and 8001 uses.
  If you want to customize them, you will need to edit the chibisafe source code to change environment variables, and then either build a local docker image to run it or run it manually.
:::

### Configuring Nginx Proxy Manager

Now, you need to configure Nginx Proxy Manager to proxy the requests to chibisafe. Go to your Nginx Proxy Manager instance and add a new proxy host.

- **Domain Names**: Your domain name (e.g. chibisafe.example.com)
- **Scheme**: http
- **Forward Hostname/IP**: \<your local ip or public ip\>
- **Forward Port**: 8001

> Don't forget to enable `cache assets` and configure an SSL certificate (also enable `force ssl`). You can use Let's Encrypt for that.

Then, head over to the location tab and add two new locations:

- **location**: /api
- **scheme**: http
- **forward hostname/ip**: \<your local ip or public ip\>
- **forward port**: 8000

And

- **location**: /docs
- **scheme**: http
- **forward hostname/ip**: \<your local ip or public ip\>
- **forward port**: 8000

You can now save the configuration and wait for Nginx Proxy Manager to apply the changes.

### DNS Configuration

Now, you need to configure your DNS to point to your server. You can use Cloudflare for that. Just add an A record pointing to your server's IP.

### Serve the uploads

As we just saw before, in the [docker-compose.yml](#installing) file we removed the static file serving. You will need to serve your chibisafe files from a different location with either a web server or a Docker container image that allows this, and then configure chibisafe accordingly.

Here is an example on how to serve the files with a docker image file server:

```yaml
version: "3.7"

services:
  sfs:
    image: halverneus/static-file-server:v1.8.3
    ports:
      - "8002:8080"
    volumes:
      - "./uploads:/web"
    environment:
      - ALLOW_INDEX=false
      - SHOW_LISTING=false

  chibisafe:
    image: chibisafe/chibisafe:latest
    environment:
      - BASE_API_URL=http://chibisafe_server:8000
    expose:
      - 8001
    restart: unless-stopped
    ports:
      - 8001:8001

  chibisafe_server:
    image: chibisafe/chibisafe-server:latest
    volumes:
      - ./database:/app/database:rw
      - ./uploads:/app/uploads:rw
      - ./logs:/app/logs:rw
    expose:
      - 8000
    restart: unless-stopped
    ports:
      - 8000:8000
```

Here is our new docker-compose.yml file that will run both chibisafe and the file server. The file server will serve the files from the `uploads` folder and listen on port 8002, so make sure to not have anything on that port. you can change the port if you want.

Now, you can run `docker-compose up -d` to start chibisafe and the file server.

Now we need to configure Nginx Proxy Manager to proxy the requests to the file server. Go to your Nginx Proxy Manager instance and add a new proxy host:

- **Domain Names**: Your domain name (e.g. files.example.com)
- **Scheme**: http
- **Forward Hostname/IP**: \<your local ip or public ip\>
- **Forward Port**: 8002

> Don't forget to enable `cache assets` and configure an SSL certificate (also enable `force ssl`).

You can now save the configuration and wait for Nginx Proxy Manager to apply the changes.

Then, head over to your chibisafe installation, as some changes must be made. Go to dashboard -> settings -> service and change the `Serve Uploads From` to your file server domain (e.g. `https://files.example.com`). Don't forget the http:// or https://!

This change may need a restart so feel free to restart your chibisafe instance by heading over to where your docker-compose.yml file is and running `docker-compose down && docker-compose up -d`.

### Done!

You can now access chibisafe through your domain name. If you have any issues, feel free to ask for help in our [Discord server](https://discord.gg/5g6vgwn).
