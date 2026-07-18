# Docker Engine Architecture

Docker CLI
Docker Deamon
containerd
containerd-shim
runC
libcontainer
namespace & CGroups

# Docker Objects
Images Networks Containers Volumes Registry

# Docker Engine Installation
docker version
docker --version
docker system info

# Service Configuration

- systemctl start docker
- systemctl status docker
- systemctl stop docker

# Docker Objects CLI

docker <docker-object> <sub-command> [options] <Arguments/Commands>

docker container run -it ubuntu
docker image build .
docker container attach ubuntu
docker container kill ubuntu
docker container ls
docker container ls -a
docker container ls -l
docker container ls -q
docker container ls -aq
docker container start 36a391532e10
docker container create httpd 
docker container start 36a391532e10
docker container run -it ubuntu
docker container ls -l
docker container exec -it b71f15d33b60 /bin/bash

# Inspectinga Container

docker container inspect webapp
docker container stats
docker container top webapp
docker container logs logtest
docker container logs -f logtest
Docker System Events
docker system events --since 60m


Linux Signals  VS Docker signels

kill –SIGSTOP 11663 $(pgrep httpd)  vs docker container pause web

kill –SIGCONT $(pgrep httpd) vs docker container unpause web
kill –SIGTERM $(pgrep httpd) vs  docker container stop web
kill –SIGKILL $(pgrep httpd) vs docker container stop web

Remove All Container
docker container ls -a
docker container ls -q

docker container stop $(docker container ls -q)
docker container rm $(docker container ls -aq)
docker container ls -a
docker container ls -q
docker container stop $(docker container ls -q)
docker container prune

# Remove Flag
docker container run --rm ubuntu expr 4 + 5

# Container Hostname
docker container run -it --name=webapp ubuntu

docker container run -it --name=webapp --hostname=webapp ubuntu

# Container – Restart Policies
docker container run --restart=unless-stopped ubuntu

# Live Restore


#  PORT mapping

docker run –p 80:5000 kodekloud/simple-webapp
docker run –p 8000:5000 kodekloud/simple-webapp
docker run –p 3306:3306 mysql
docker run –P --expose=8080 kodekloud/simple-webapp


# Troubleshoot Docker Daemon
journalctl -u docker.service

# Daemon Configuration File
/etc/docker/daemon.json

# Debug in Docker
docker system info

# Logging Drivers
docker run –d --log-driver json-file nginx

# image Build
Dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install python
RUN pip install flask
RUN pip install flask-mysql
COPY . /opt/source-code
ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run 

docker build . -t my-custom-app

# Build Context

docker build . -t my-custom-app
docker build /opt/my-custom-app

# Dockerfile
docker build -f Dockerfile.dev https://github.com/myaccount/myapp

# docker ignore file
.dockerignore

# Build Cache
- Cache Busting
- Version Pinning

Multi - Stage Builds

# Persist State 
 stateless and statefull applications


# volumes:

use /Users/ziyadtarek/Desktop/iti-docker-2026/README.md

# networks :
use /Users/ziyadtarek/Desktop/iti-docker-2026/README.md


Docker compose
/Users/ziyadtarek/Desktop/iti-docker-2026/README.md