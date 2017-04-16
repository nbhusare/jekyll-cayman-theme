---
layout: default
title:  "Docker Troubleshooting tip"
date:   2017-04-12
categories: main
---

If you are using 64bit Windows 10 Pro or Enterprise or Education editions, the recommended way to install Docker is to use [Docker for Windows](https://www.docker.com/docker-windows). On older version of Windows or Windows Home edition, you'll have to use [Docker Toolbox](https://www.docker.com/products/docker-toolbox). 

Docker Toolbox will install following tools - Docker CLI, Docker Machine, Docker Compose, Kitematic, Docker QuickStart shell and Oracle VM VirtualBox. You can right away start using the Docker Quick Start shell to execute the docker commands. If you try using shells like bash or cmd or powershell, there are good chances that you'll see the following error

```
SET DOCKER_TLS_VERIFY=1 SET DOCKER_HOST=tcp://192.168.99.100:2376 SET DOCKER_CERT_PATH=C:\Users\neepal.docker\machine\machines\default SET DOCKER_MACHINE_NAME=default SET COMPOSE_CONVERT_WINDOWS_PATHS=true REM Run this command to configure your shell: REM @FOR /f "tokens=*" %i IN ('docker-machine env default') DO @%i
```

This is because your Docker client environment is not set. Command *`docker-machine env [OPTIONS] [arg...]`* will display the commands to set up the required environment. PS [docker-machine env](https://docs.docker.com/machine/reference/env/) for more details

* **Bash** : *docker-machine env --shell bash default*
```export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="C:\Users\neera\.docker\machine\machines\default"
export DOCKER_MACHINE_NAME="default" 

OR 

eval "$(docker-machine env dev)"
```

* **cmd.exe** : *docker-machine env --shell cmd default*
```SET DOCKER_TLS_VERIFY=1
SET DOCKER_HOST=tcp://192.168.99.100:2376
SET DOCKER_CERT_PATH=C:\Users\neera\.docker\machine\machines\default
SET DOCKER_MACHINE_NAME=default
SET COMPOSE_CONVERT_WINDOWS_PATHS=true
```

* **PowerShell** : *docker-machine env --shell powershell default*
```$Env:DOCKER_TLS_VERIFY = "1"
$Env:DOCKER_HOST = "tcp://192.168.99.100:2376"
$Env:DOCKER_CERT_PATH = "C:\Users\neera\.docker\machine\machines\default"
$Env:DOCKER_MACHINE_NAME = "default"
$Env:COMPOSE_CONVERT_WINDOWS_PATHS = "true"
```

After setting the above commands, all your docker commands will run against the Docker host.
