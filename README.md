# Docker Setup for Multiple Angular Projects

This repository demonstrates how to deploy multiple Angular projects using Docker containers with Nginx as a web server. This setup allows for each Angular project to run in its isolated container, making it a practical solution for managing and deploying micro-frontends.

## Prerequisites

Make sure you have the following installed:

- Node.js
- Angular CLI (`ng`)

## Create the Angular projects

The main Angular project will act as a wrapper for the sub-projects.

Create the main (outer) project first:

```bash
ng new main-project --create-application="false"
```

Create the sub-projects:

```bash
ng new project-1
```

And:

```bash
ng new project-2
```

### Move the sub-projects into main

Move the sub-projects under the main project folder:

```bash
mv project-1 main-project && mv project-2 main-project
```

## Docker

The `docker-compose.yml` file defines the services, networks, and volumes for the Docker containers.

Here is a sample configuration:

### Docker compose

```yml
version: "3.8"

networks:
  test_network:
    driver: bridge

services:
  project-1:
    build:
      context: .
      dockerfile: Dockerfile.project1
    container_name: project-1
    ports:
      - "81:80"
    networks:
      - test_network
    command: ["nginx", "-g", "daemon off;"]

  project-2:
    build:
      context: .
      dockerfile: Dockerfile.project2
    container_name: project-2
    ports:
      - "82:80"
    networks:
      - test_network
    command: ["nginx", "-g", "daemon off;"]
```

### Dockerfiles

The Dockerfile is used to set up the Angular environment, build the Angular project, and serve it using Nginx.

Here's how to set up the Dockerfile for `project-1`:

```
FROM nginx:stable-alpine3.17 AS base
ENV PROJECT_NAME=project-1

# Setup the Angular project
WORKDIR /usr/local/app
COPY package*.json /usr/local/app/
COPY angular.json /usr/local/app/
COPY ./projects/${PROJECT_NAME} /usr/local/app

# Update packages and install NodeJS
RUN apk add --update nodejs npm && \
    npm install @angular/cli@16.2.1 -g && npm ci && \
    npm run build ${PROJECT_NAME}

RUN npx ngcc --properties es2015 browser module main --first-only --create-ivy-entry-points

# Erase default Nginx content and add dist files from Angular
RUN rm -rf /usr/share/nginx/html && \
    mkdir -p /usr/share/nginx/html && \
    # echo '<h1>test</h1>' >> /usr/share/nginx/html/index.html
    cp -r /usr/local/app/dist/${PROJECT_NAME}/* /usr/share/nginx/html/

EXPOSE 81
```

Repeat the same process for `project-2`, but make sure to change the `PROJECT_NAME` and exposed port:

```
FROM nginx:stable-alpine3.17 AS base
ENV PROJECT_NAME=project-2
...
...
```

Change the port being exposed by the second project's container:

```
...
...
EXPOSE 82
```

The `--properties es2015 browser module main --first-only --create-ivy-entry-points` flags are used to specify which properties of the packages should be processed, in this case:

`es2015`: This flag indicates that the packages should be processed to provide compatibility with ES2015 (ECMAScript 2015) standards. It transforms code to a version that is more compatible with older JavaScript engines.

`browser`: This flag indicates that the browser-specific version of the package should be generated. This helps to ensure compatibility with browser environments.

`module`: This flag specifies that the package should be processed for compatibility with module systems.

`main`: This flag indicates that the "main" entry point of the package should be processed.

`--first-only`: This flag tells ngcc to only process the first entry point, which can speed up the process if multiple entry points are available.

`--create-ivy-entry-points`: This flag creates Ivy entry points for compatibility. Ivy is the new rendering engine for Angular, and this flag ensures that the packages are compatible with Ivy.

## Nginx files

Here's an example Nginx server block you can use for development:

```
server {
	listen 80;
	server_name localhost;
	index index.html index.htm;
	text/xml application/xml application/xml+rss text/javascript;

	# gzip on;
	# gzip_min_length 1000;
	# gzip_proxied expired no-cache no-store private auth;
	# gzip_types text/plain text/css application/json application/javascript application/-javascript

	location /project-1 {
		proxy_pass http://localhost:81;
        proxy_set_header HOST $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
	}

	location /project-2 {
		proxy_pass http://localhost:82;
        proxy_set_header HOST $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

### Production Nginx server block

Here's an example production server block file for the host's Nginx service to join the projects under one domain:

```
upstream project_1 {
    server localhost:81;
}

upstream project_2 {
    server localhost:82;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location /project-1 {
        proxy_pass http://project_1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /project-2 {
        proxy_pass http://project_2;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**NOTE:** The above configuration assumes you've set up SSL certificates using Let's Encrypt.

## Build the Docker containers

Build the containers

```bash
docker-compose build
```

Then start the containers:

```bash
docker-compose up
```

You can specify a different Docker Compose file using the `-f` flag:

```bash
docker-compose -f another-docker-compose.yml up
```
