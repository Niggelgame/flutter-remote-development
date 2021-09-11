# Remote Flutter Web Development with Docker and Code-Server

This is a small explanation on how to develop [Flutter](https://flutter.dev/) inside of a Docker-Container with [Code-Server](https://github.com/cdr/code-server). I created this, to be able to do Flutter development on my iPad (Just open the two tabs in Split-Screen).

![Working Screenshot](screenshot.png?raw=true "Screenshot")

## Setup Linux Server

You need to have [Docker](https://www.docker.com/) installed. (If not, take a look [here](https://docs.docker.com/get-docker/).)

Please continue by installing `nginx`, if not installed yet:

For Debian-based systems:
```bash
sudo apt update
sudo apt install nginx
sudo systemclt start nginx
sudo systemctl enable nginx
```

For Arch-based systems:
```bash
pacman -Syu
pacman -S nginx
sudo systemclt start nginx
sudo systemctl enable nginx
```

## Create needed files

Setup Code-Server-Config (You can try to ignore this step, since the config will be generated most of the time - but I had some errors while doing so):

In your home-directory, run:

`nano .config/code-server/config.yaml`

This will open the configuration file for code-server.
Paste this text inside:
```yaml
bind-addr: 127.0.0.1:8080
auth: password
password: <Your password>
cert: false
```
Set a password instead of `<Your password>`. (If you let code-server generate your config files, it will generate a pretty strong password)

### Dockerfile

In the next step, create the Dockerfile:

`nano Dockerfile`

and paste

```Dockerfile
FROM codercom/code-server:latest

# Install Flutter
RUN sudo apt-get update
RUN sudo apt-get install -y unzip
# Usually in a normal Linux environment, you should not do that
RUN sudo chmod 777 /opt
RUN git clone https://github.com/flutter/flutter.git -b stable --depth 1 /opt/flutter
ENV PATH="/opt/flutter/bin:${PATH}"
RUN flutter doctor
RUN flutter config --enable-web

# Install Code extensions
RUN code-server --install-extension dart-code.flutter
```

Since web is available on `stable` now, you can use it too. To use more experimental features, consider setting the branch to something *higher than stable* ([Branch/Channel descriptions](https://github.com/flutter/flutter/wiki/Flutter-build-release-channels))

### Setup your startup and stopping command

Create a file called something like *start-code-server* and edit to:

```bash
#/bin/sh

# This will start a code-server container and expose it at http://127.0.0.1:8081 and http://127.0.0.1:8082.
# It will also mount your current directory into the container as `/home/coder/project`
# and forward your UID/GID so that all file system operations occur as your user outside
# the container.
#
# Your $HOME/.config is mounted at $HOME/.config within the container to ensure you can
# easily access/modify your code-server config in $HOME/.config/code-server/config.json
# outside the container.

# Create links for nginx to create the reverse proxy
sudo ln -s /etc/nginx/sites-available/code-server.conf /etc/nginx/sites-enabled/code-server.conf
sudo ln -s /etc/nginx/sites-available/code-server-flutter.conf /etc/nginx/sites-enabled/code-server-flutter.conf
sudo systemctl restart nginx

mkdir -p ~/.config
docker run -d -it --name code-server -p 8081:8080 \
  -p 8082:8081 \
  -v "$HOME/.config:/home/coder/.config" \
  -v "*<Your Project Directory>*:/home/coder/project" \
  -u "$(id -u):$(id -g)" \
  -e "DOCKER_USER=$USER" \
  custom-code:latest
```
Replace `*<Your Project Directory>*` with the directory on your server/machine you want to expose to the code-server-container. I usually have "${HOME}" set there, since I will be able to open every subdirectory with `Ctrl + K   Ctrl + O` or `F1` with `Open Folder`.

To be able to execute it, run 

`chmod +x start-code-server`

### Build Container

In the directory of your `Dockerfile` create another file called `build-image`. 
Paste:

```bash
# /bin/bash

docker build . -t custom-code:latest
```

and do 

`chmod +x build-image again`.
Then run `./build-image`. It may take some time.


### Setup nginx

I wanted to have a reverse proxy from 2 different subdomains of mine to point to my code-server and my flutter-web result.

Therefore I generated the SSL-certificates with [certbot]("https://certbot.eff.org/").

Next, create two files in `/etc/nginx/sites-available/`: `code-server.conf` and `code-server-flutter.conf`.

For `code-server.conf`, paste this code:

```conf
server_tokens off;

server {
    listen 80;
    server_name *<domain>*;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name *<domain>*;

    access_log /var/log/nginx/code-server.app-access.log;
    error_log  /var/log/nginx/code-server.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    # SSL Configuration (Only if you have the certificates)
    ssl_certificate /etc/letsencrypt/live/*<domain>*/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/*<domain>*/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
```
Replace *\<domain>* with your domain/subdomain for your code-server environment.


For `code-server-flutter.conf`, paste this code:

```conf
server {
    listen 80;
    server_name *<flutter-domain>*;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name *<flutter-domain>*;

    access_log /var/log/nginx/code-server-flutter.app-access.log;
    error_log  /var/log/nginx/code-server-flutter.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    # SSL Configuration (Only if you have the certificates)
    ssl_certificate /etc/letsencrypt/live/*<flutter-domain>*/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/*<flutter-domain>*/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
```

Replace *\<flutter-domain>* with your domain/subdomain for your Flutter-Web result environment.

### Remove nginx proxy

Me not liking to have the proxy turned on all the time, I have created a small script to delete the config-links in a second:
Just execute

```bash
sudo rm /etc/nginx/sites-enabled/code-server.conf
sudo rm /etc/nginx/sites-enabled/code-server-flutter.conf


sudo systemctl restart nginx
```


## Start developing

To start your Docker-Dev environment, run your `start-code-server` file.


Now you should be able to go to your domain specified in the *Setup nginx* step and see your code-server.

To run your Flutter project, run

`flutter pub get`

and 

`flutter run -d web-server --web-hostname=0.0.0.0 --web-port=8081`

It will expose itself to the domain specified for the flutter-result.


## Questions

If you have any questions, feel free to contact me at this [email](mailto:nicolas@kedil.de?subject=[GitHub]%20Flutter%20Docker%20Server) or by creating a issue.

You will find all the files mentioned in the repository.
