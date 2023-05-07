---
layout: post
title: "Using Nginx and Gunicorn to Run a Django based Application"
date: 2023-04-24 11:45:00 +0000
tags: servers
description: "An Nginx front-facing server with a Gunicorn backend serving a Django application"
---
# Introduction
- In this post, I will illustrate how to create a production-ready Django-based application using the Nginx web server and Gunicorn WSGI server. A  high-level overview of this implementation is as follows:

![](/assets/post_images/wsgi_web_highlevel.png)

- As illustrated, to enable portability, the entire application has been set up using containers. Due to this containerization, re-deploying the application in another host is as simple as copying the application base directory and running the containers.

- The main functions of using a Nginx server can be given as follows:
  - Web server: Nginx can serve static and dynamic content over HTTP and HTTPS protocols. It is optimized for handling high traffic and concurrent connections, making it ideal for serving large-scale websites and applications.
  - Reverse proxy: Nginx can act as a reverse proxy, which means it can handle incoming requests from clients and forward them to backend servers or services. This can help to improve the performance and scalability of web applications.
  - Load balancer: Nginx can distribute incoming traffic across multiple backend servers or services, which can help to improve the availability and reliability of web applications. It can use various load balancing algorithms to evenly distribute traffic.
  - HTTP cache: Nginx can cache frequently requested content in memory or on disk, which can help to reduce the load on backend servers and improve the response time for clients.
  - Security: Nginx can be configured to enhance the security of web applications by implementing various security features such as SSL/TLS encryption, HTTP authentication, access control, and DDoS protection.

- One of the main advantages of using a WSGI server is concurrency. A WSGI server runs a separate instance of the web application or framework, allowing multiple requests to be handled concurrently.

- We use the following file system in this work. 

```
database
nginx
    Dockerfile
    nginx.conf
    <ssl certificate file>.pem
    <ssl certificate key file>.key
wsgi
    Dockerfile
    requirements.txt
    django_app
        static
        main_app
            __init__.py
            asgi.py
            wsgi.py
            settings.py
            urls.py
        ......
        manage.py
db_backup.sh
docker-compose.yml
run_docker_dev.sh
run_docker_prod.sh
```
- There are three containers in our application. They are as follows:
  1. Web server
  2. WSGI server
  3. Postgresql database

- To deploy this application, we run `docker-compose.yml` which can be illustrated as follows:

```
version: '3.9'

services:
  wsgi:
    container_name: wsgi
    restart: always
    build: ./wsgi
    ports:
      - "<host-accessible wsgi port>:<wsgi container port>"
    deploy:
      resources:
        reservations:
          devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
    command: gunicorn -w <number of workers. ex: 5> -b 0.0.0.0:<wsgi container port> main_app.wsgi:application --chdir 'django_app'
    depends_on:
      - projdb

  nginx:
    container_name: nginx
    restart: always
    build: ./nginx
    ports:
      - "<host-accessible nginx port>:<nginx container port>"
    depends_on:
      - wsgi
    volumes:
      - type: bind
        source: ./wsgi/django_api/static
        target: /app/static

  projdb:
    image: postgres:latest
    volumes:
      - type: bind
        source: ./database
        target: /var/lib/postgresql/data/
    environment:
      - "POSTGRES_USER=<username>"
      - "POSTGRES_PASSWORD=<password>"
      - "POSTGRES_DB=<DB name to be used with Django app>"
```

- In the above file, it can be noted that there's a bind volume that binds the `database` in the host to the database container's `/var/lib/postgresql/data/` directory so that any change is synced between two directories in real time and the change is persistent after the containers are stopped. This will enable a convenient way for database backup.

- Furthermore, it should be noted that all the static files of the Django app should be placed in `wsgi/django_app/static` directory.

- Now let us take a look at the configuration files for WSGI and Nginx containers in more detail. 

# Nginx server configuration

- For the Nginx webserver, the `nginx` directory contains the docker file to start the Nginx container and the configuration file for the Nginx web server. The nginx configuration file (`nginx.conf`) can be given as follows: 

```
# Define an upstream group of backend servers for load balancing (list the backend Django servers here)
upstream backend_servers {
    server 127.0.0.1:8000;
}

# Configure the server block to listen on port 443 for HTTPS traffic
server {
    listen 443 ssl;
    server_name <base URL of the website without https:// prefix>;

    # Specify SSL certificate and key for HTTPS encryption
    ssl_certificate /app/<ssl certificate file>.pem;
    ssl_certificate_key /app/<ssl certificate key file>.key;

    # Proxy incoming requests to the backend servers using load balancing
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Serve static files with caching headers for improved performance
    location /static/ {
        alias /app/static/;
        expires 1d;
        add_header Cache-Control "public";
    }
}
```
- In the above file, `backend_servers{}` block defines all the backend Django server instances. Here, we have assumed that only one Django server is running at port `8000`.
- The Nginx server will distribute the incoming requests across the backend Django servers.
- We have also assumed here that the client connects with the Nginx server using `https`. Therefore, the SSL certificate and SSL certificate keys should be provided in the file.
- Additionally, the URL of the website also needs to be provided.
- The docker file for the Nginx server is as follows:

```
# Get the base image
FROM nginx

# Copy nginx configuration file to the container
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Set the working directory
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Run nginx
CMD ["nginx", "-g", "daemon off;"]
```
- This file simply starts a container, mounts the Nginx configuration file in the Nginx container, set up the working directory and starts an Nginx process.
- Now let us look at the Wsgi configuration files.

# WSGI server configuration
- The docker file for Nginx server is as follows:

```
# Get the base image 
FROM nvidia/cuda:12.1.1-cudnn8-devel-ubuntu20.04

# Install necessary dependancies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3-pip

# Set the working directory
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
```

- This file starts a container, installs necessary Linux packages, set the working directory, and finally installs Python packages stated in the `requirements.txt`.
- There are also some important configurations that need to be set up in the `settings.py` file in `wsgi/django_app/main_app`. They are as follows:

```
# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = False

STATIC_URL = 'static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

- Here, we have setup the base directory, set debug mode to false, and set the static directory.
- Then in `urls.py` in `wsgi/django_app/main_app`, add the following lines:

```
urlpatterns = [
    .............,
    .............,
] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

- The above lines instruct Django to serve static files from the `STATIC_ROOT` directory at the `STATIC_URL` endpoint.

# Final remarks
- In a nutshell, we only need to run `docker_run_rpod.sh` or `docker_run_dev.sh` (for development purposes with attached mode to view logs) to run our application. These files are as follows:
  - `run_docker_dev.sh`:

  ```
  echo killing old docker processes
  docker-compose rm -fs

  echo building docker containers
  docker-compose up --build --remove-orphans
  ```

  - `run_docker_prod.sh`:

  ```
  echo killing old docker processes
  docker-compose rm -fs

  echo building docker containers
  docker-compose up --build -d --remove-orphans
  ```

- Also note that these configurations have been tested on `docker-compose version 1.29.2`.
- To view the status of the docker containers, use the command: `docker-compose ps` from the base directory.
- To remove all containers created by the docker-compose.yml file, use the command: `docker-compose rm -fs` from the base directory.
- After any modifications to the Django app, run the following two commands from the base directory when containers are running to ensure that the changes take effect:
  - `docker-compose exec wsgi sh -c "cd django_app && python3 manage.py makemigrations"`
  - `docker-compose exec wsgi sh -c "cd django_app && python3 manage.py migrate"`
- If you need to backup a database to a SQL file, run `db_backup.sh` while the containers are running. The file can be given as follows:

  ```
  docker exec <database container name> pg_dump -U <user name> -d <database name> > <backup file name>.sql
  ``` 