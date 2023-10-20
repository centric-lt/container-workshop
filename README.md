# Workshop

## Installing container runtime

Below install assumes that VSCode is used as a Docker development environment. Adjust instructions appropriately if another editor is used.

Example commands are expected to be run in this repository root folder.

### Docker Desktop on Windows plus WSL2

For personal use Docker Desktop is free and provides good and easy to setup integration with "VS Code" editor. Instructions can be [found here:](https://docs.docker.com/desktop/windows/wsl/) and [here:](https://code.visualstudio.com/docs/remote/containers-tutorial).

### Docker Desktop on Linux

For personal use Docker Desktop is free and provides good and easy to setup integration with "VS Code" editor. Instructions can be [found here:](https://docs.docker.com/desktop/install/linux-install/) and [here:](https://code.visualstudio.com/docs/remote/containers-tutorial).


### Docker Engine on Windows and WSL2

Usage of Docker Desktop is restricted in enterprise environments, therefore alternative setup is needed for such Windows environments.

1. Install WSL2 with Ubuntu
    * (https://ubuntu.com/tutorials/install-ubuntu-on-wsl2-on-windows-10#1-overview)
    * Use version 20.04.
2. Install Docker on Windows
    * (https://dev.to/felipecrs/simply-run-docker-on-wsl2-3o8)
3. Integrate with WSL2
    * (https://securecloud.blog/2021/12/07/wsl2-use-docker-with-vscode-without-docker-desktop/)


### Docker Engine and Linux

Docker Engine can be directly used from  VSCode on Linux environment due to container engine and editor will run on within the same OS scope.

1. Install Docker runtime (docker-ce), using yum, apt, zipper or any native package manager.
    * Alternatively Docker engine installation can be [followed](https://docs.docker.com/engine/install/).
2. Install VSCode as defined VS Code Linux [setup page](https://code.visualstudio.com/docs/setup/linux).
3. Install Docker Extension in VS Code as [defined here](https://code.visualstudio.com/docs/containers/overview).

Docker engine can be used via Linux terminal application or from within VSCode Terminal subwindow.

## Manipulating images and containers via Docker

To build new docker image manually, run the following commands.

```bash
docker build -t app -f Dockerfile-app .
docker build -t train -f Dockerfile-train .
```

To start up the container, you need to explicitly specify image, networking, volume and other parameters on command line.

```bash
# Run and detach from terminal
docker run -d -p 127.0.0.1:8081:5000/tcp --name training-app train
```

List active containers using `ls` and terminate them by name using `kill`.

```bash
docker ps
docker ps -a
docker kill train-app
```

Listing docker images and image clenup. Image clean up is needed to be done from time to time to reclaim space, used by old/unused image versions. Note, that clean up will most likely increase next build process as some build dependency images might need to be downloaded.

```bash
docker image list
docker image prune
docker image list
```

## Working with Dockerfile

Container build is based on initial base image, specified in FROM definition. If image name is specified implicitly, then search is done in default public repository, usually `docker.io`. If an explicit name is used, then image is pulled from specified repository.

When searching for an image on https://hub.docker.com/, https://quay.io/search or other public repository, it is important to take into account image publisher. For example search for "python" results in more than "80000" hits. There have been cases of malicious content in Docker images in public repositories so a trusted publisher should be used.

It is advisable to minimize number of RUN commands in the Dockerfiles intended for production use. This allows to reduce image size due to reducing need to do multiple changes for the same FS block and smaller layer diff.

EXPOSE instruction in Dockerfile does not open ports automatically, when running container. When run with -P option, docker will pick all EXPOSE ports on random high ports. Ports can be explicitly override using -p option.

CMD instruction defines container workload application and its parameters by default. It can be overriden at run time.

## Working with Compose

TLDR version. Requires correct Compose file, though. :) Run the following command in the folder, where `docker-compose.yaml` does reside. Compose will parse file, fetch/build images and start applications. Access the application using  http://127.0.0.1:8080/qod and get your Quote Of the Day.

**NOTE:** Due to public access to Quote of the day service was closed more than year ago, there no actual quotes, unfortunately. Free service allows to get only 5 quotes per day using personal auth token and implementing this was considered a bit out of scope for Compose demo.

```bash
# To run in the foreground and see logs in terminal
docker-compose up
# To disconnect from terminal and have available console
docker-compose up -d
# When finished with application stop it and dismantle networks
# Otherwise next example will fail as it will not be able to bind to 8080 port
docker-compose down
```

While volumes can be defined explicilty using docker CLI, it is more convienient to define them in Docker Compose file. This is done in a `docker-compose-volumes.yaml` file, which has volume definitions added like the following
```yaml
# Specify volume or local path and define mount point in container
volumes:
      - .:/app
      - logvolume:/var/log

# Define named volume
volumes:
   logvolume: {}
```

As the file name for volume usage example is not default, it needs to be explicitly specified. Also it is recommended to specify distinct project name (`-p volume-example`) as by default Docker assigns project name according to parent folder. In some cases running multiple Compose applications in the same project scope leads to unexpected and undesired results, like "orphaned containers found".

```bash
docker-compose -f docker-compose-volume.yaml -p volume-example up -d
```

Binding multiple applications in compose file is done by defining container names and their dependencies. Note, that these names only define relations within YAML definition, not actual container and/or DNS names. For that, names needs to be set explicitly in Compose file. However in this case a developer is responsible for the name uniquiness. This might be needed, when linking multiple Compose applications via network as containers within same Compose are accessible via localhost.

```yaml
    my_app:
        container_name: apache-__app_server_username__
        hostname: apache-__app_server_username__
```
## Runing Docker in Docker

In certain cases there might be a need to isolate Docker container, within container, i.e. Docker in Docker (DinD). Such inception allows completely isolate container environment from host. It is doable, however this imposes a set of issues:
* Networking for applications becomes more complex and error prone.
* Container build environment becomes more complicated as build context need to be provided to containerized Docker "host".
    * This can be mitigated by locally building containers and pushing them to repo, but that mostly defeats  purpose of DinD.
* To avoid serious performance penalties volumes should be defined and mounted in multiple levels to avoid writing to Copy On Write layered FS, used in containers.

Such example is defined in `docker-compose-dind.yaml` file.

```bash
docker-compose -f docker-compose-dind.yaml -p dind-example up -d
```

If checked, you should see only two docker containers running in the system. However if container list on `docker` "host" is verified, then actual payload will be seen.

```bash
:~/git/container-workshop$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                                      NAMES
986ac52e9f9d   docker:dind   "dockerd-entrypoint.…"   16 minutes ago   Up 16 minutes   2375-2376/tcp                                              container-workshop-service-1
4f1632c0a562   docker:dind   "dockerd-entrypoint.…"   16 minutes ago   Up 16 minutes   2375-2376/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   container-workshop-docker-1
:~/git/container-workshop$ docker-compose -f docker-compose-dind.yaml -p dind-example ps
NAME                           COMMAND                  SERVICE             STATUS              PORTS
container-workshop-docker-1    "dockerd-entrypoint.…"   docker              running             2375-2376/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
container-workshop-service-1   "dockerd-entrypoint.…"   service             running             2375-2376/tcp
:~/git/container-workshop$ docker-compose -f docker-compose-dind.yaml -p dind-example exec -it docker docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                    NAMES
e9275f041586   app            "python3 app.py"         16 minutes ago   Up 16 minutes   0.0.0.0:8080->8080/tcp   app-app-1
b7ef72934d7c   redis:alpine   "docker-entrypoint.s…"   16 minutes ago   Up 16 minutes   6379/tcp                 app-redis-1

```

To summarize, usually it might be more efficient to design volume structure properly and isolate container files within volumes. This might be more efficient than DinD implementation, which will still require multilevel volume mounts. Usually such isolation is used on application build hosts to ensure clean build environment and avoid situation, when leftovers from previous build might affect new build and/or test statges.

## More examples from Docker

More complex application examples from the Docker example/demo [application repository](https://github.com/dockersamples).

Folder `k8s-wordsmith-demo` contains [K8s Wordsmith Demo](https://github.com/centric-lt/container-workshop/blob/master/k8s-wordsmith-demo/README.md) application, which selects 5 random words from random word sources, running in separate containers and presents a constructed sentence.

A simple [NGinx proxy for Flask application with MySQL backend](https://github.com/centric-lt/container-workshop/blob/master/nginx-flask-mysql/README.md) can be found in `nginx-flask-mysql` folder.
