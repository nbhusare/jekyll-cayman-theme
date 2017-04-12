---
layout: default
title:  "Docker Troubleshooting tip"
date:   2017-04-12
categories: main
---
While working with Docker, you might come across the following error. What this means is that - 

error during connect: Get http://%2F%2F.%2Fpipe%2Fdocker_engine/v1.26/containers/json: open //./pipe/docker_engine: The system cannot find the file specified. In the default daemon configuration on Windows, the docker client must be run elevated to connect. This error may also indicate that the docker daemon is not running.

The solution is to run `docker-machine env default`. This will list down following commands that when run fixes the issue

SET DOCKER_TLS_VERIFY=1
SET DOCKER_HOST=tcp://192.168.99.100:2376
SET DOCKER_CERT_PATH=C:\Users\neepal\.docker\machine\machines\default
SET DOCKER_MACHINE_NAME=default
SET COMPOSE_CONVERT_WINDOWS_PATHS=true
REM Run this command to configure your shell:
REM     @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i
