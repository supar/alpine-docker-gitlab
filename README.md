## Alpine Gitlab Docker

This project is a fork of the [alpine-docker-gitlab](https://github.com/alpinelinux/alpine-docker-gitlab) repository. The main objective of this fork is to have PostgreSQL, and Nginx as extenal services, they will be made available either on the host machine or remotely.

# Alpine Linux based docker image and tools for Gitlab.

## Why another Gitlab docker image?

 - Completely based on Alpine Linux (no static binaries)
 - Use separate docker images for services (where possible)
 - Optimized for size
 - Bundle services with docker compose

## Build and prepare images

There is useful tool to build - [task](https://taskfile.dev)

```bash
task build
```

## Setup clean instalation

To get GitLab up and running, you will need to create an [.env](https://docs.docker.com/compose/env-file/) file to define the required environment variables. This file should be placed alongside your docker-compose file.

```
GITLAB_ROOT_PASSWORD=gitlab_admin_password

POSTGRES_PASSWORD=postgres_password
POSTGRES_USER=postgres_user
POSTGRES_HOST=postgres_host
POSTGRES_DB=postgres_db_name
```

Next, place there the GitLab infrastructure roots by specifying the relevant paths for storing repositories, configuration files, and logs in your docker-compose file:

```
GITLAB_STORAGE_ROOT=/path/to/repositories
GITLAB_CONFIG_ROOT=/path/to/store/all/configs
GITLAB_LOG_ROOT=/path/to/logs
```

Classic schema

```
GITLAB_STORAGE_ROOT=/var/lib/gitlab
GITLAB_CONFIG_ROOT=/etc/gitlab
GITLAB_LOG_ROOT=/var/log/gitlab
```

Bring up the containers

```docker-compose up```

Watch the output on console for errors. It will take some time to generate the db
and update permissions. Ones its done without errors you can Ctrl+c to stop the
containers and start them again in the background.

## Access the application

Visit your Gitlab instance at http://dockerhost

## Configuration

The default configuration is very limited and requires to be improved.

### Short how-to for reverse proxy
Let's configure the domain and enable TLS. For TLS, we'll use an ideal solution in my opinion, the [ACME](https://github.com/acmesh-official/acme.sh) utility. I know that when reading documentation, it's tempting to blindly copy configuration examples. You can do the same here, but there are a few things to pay attention to:

- upstream points to gitlab-workhorse, there nginx will forward requests, the port forwarded from the puma service container;
- WEBROOT configuration for acme (`/etc/nginx/acme.conf`);
- unlimit request body.

```
upstream gitlab-workhorse {
  # GitLab socket file,
  # for Omnibus this would be: unix:/var/opt/gitlab/gitlab-workhorse/sockets/socket
  server 127.0.0.1:8181 fail_timeout=0;
}

## Normal HTTP host
server {
    listen 80;
    server_name gitlab.domain;

    include /etc/nginx/acme.conf;

    location / {
        return 301 https://$server_name$request_uri;
    }
}

## Normal HTTP host
server {
    listen 443 ssl;
    server_name gitlab.domain;

    ssl_certificate /path/fullchain.cer;
    ssl_certificate_key /path/gitlab.domain.key;

...

    location / {
      client_max_body_size 0;
      gzip off;

      ## https://github.com/gitlabhq/gitlabhq/issues/694
      ## Some requests take more than 30 seconds.
      proxy_read_timeout      300;
      proxy_connect_timeout   300;
      proxy_redirect          off;

      proxy_http_version 1.1;

      proxy_set_header    Host                $http_host;
      proxy_set_header    X-Real-IP           $remote_addr;
      proxy_set_header    X-Forwarded-Ssl     on;
      proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
      proxy_set_header    X-Forwarded-Proto   $scheme;
      proxy_set_header    Upgrade             $http_upgrade;
      proxy_set_header    Connection          $connection_upgrade_gitlab;

      proxy_pass http://gitlab-workhorse;
    }
}
```

### Enabling TLS

I preferred to create a separate user for obtaining and processing certificates.

```bash
useradd -m acmeuser
```

1. Configure `/home/acmeuser/.acme.sh/acme.sh.env.`

```
export LE_WORKING_DIR="/home/acmeuser/.acme.sh"
alias acme.sh="/home/acmeuser/.acme.sh/acme.sh"

export CERT_HOME=
export ACCOUNT_EMAIL=
export WEBROOT=
```

2. Create file `/etc/nginx/acme.conf` and includes its

```
# server discovery for the ACME
location ^~ /.well-known/acme-challenge/ {
    allow all;
    default_type "text/plain";

    root /usr/share/nginx/html;
}
```

3. Test nginx configuration `nginx -t` and restart its
4. It's time to issue the certificate. Do it as `acmeuser`

```
acme.sh --issue -d gitlab.domain --webroot $WEBROOT --log --debug
```

The command includes the debug option to help troubleshoot any potential issues. In case of success, you will receive a detailed report with the location where the certificates are saved. This information needs to be added to the TLS section of the Nginx configuration for our domain (server { .. }).

### Backups

To exclude some items from the backup you can set the environment variable
`$GITLAB_BACKUP_SKIP` which will set `SKIP=` see:
https://docs.gitlab.com/ee/raketasks/backup_restore.html

