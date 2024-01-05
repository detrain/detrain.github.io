---
title: Docker Basic Commands
date: 2023-12-21 19:56:51 -0600
categories: [Docker]
tags: [docker]     # TAG names should always be lowercase
---
> **DISCLAIMER**: This blog follows along with Bret Fisher's **Docker** Udemy course, certain areas dive deeper than the course, while other parts I may gloss over. I wanted a place to document all of the fun use cases of Docker as referencing the videos became a bit tedious and figured my own interpretation serves as a good reference for future me and hopefully any random internet strangers that take interest!
{: .prompt-info }


This post introduces the basic usage of docker. I assume you have docker installed, and you have some basic knowledge of the [technology](https://docs.docker.com/guides/get-started/).


#### Help Flag
Say hello to your best friend [Moby Dock](https://www.docker.com/blog/call-me-moby-dock/), just kidding it is `--help`! This **Docker** flag proves useful if you forget what command you need to run, or an option you may want to include with a command.

```bash
detrain@detrain:~$ docker --help
```

## Basic Commands
Below are basic commands one may find useful for basic image/container management. Not the most fun post, but we need a baseline so that one knows how to perform basic operations on an image or container. Furthermore, I do not intend to explain all of the commands. I encourage you to append the good ole `--help` to assist you, if needed.


#### pull
`docker` defaults to its official registry, [docker hub](https://hub.docker.com/). An awesome tool to pull images to your local machine.

```bash
detrain@detrain:~$ docker pull nginx
```


#### image
`docker image` enables us to manage the images on the local machine.

```bash
detrain@detrain:~$ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        latest    d453dd892d93   8 weeks ago   187MB
```


#### container
`docker container` enables us to manage the containers on the local machine.

```bash
detrain@detrain:~$ docker container ls
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS     NAMES
fc105ce46855   nginx     "/docker-entrypoint.…"   38 seconds ago   Up 37 seconds   80/tcp    modest_shirley
```

Here are some tips for container management:
- `docker container ls`: shows all active containers
- `docker container ls -a`: shows all containers
- `docker container rm <ID|name>`: remove a stopped container
- `docker container <start|stop> <ID|name>`: start/stop a container

> I have seen those unfamiliar with docker run a plethora of containers and fail to clean them up. Then they wonder why they have no space left on disk. The first things I always ask them is show me the output of `docker container ls -a` and most of the time they have an **ABUNDANCE** of stopped containers that need to be removed!
{: .prompt-tip }

#### run
`docker run` leverages the aforementioned commands to activate a container for an image.

```bash
detrain@detrain:~$ docker run -d --rm nginx
f1ffad699bf82694ec817487af53baacb4c7106d3a5d369fa0883b3155c8304d
```

Here are some quality options associated with the common command run:
- `-d`: runs the container in a detached state (run in background)
- `--rm`: remove the container on exit (one may not desire this, but overall quality flag to avoid container bloat)
- `-p <host:container>`: publishes a container port to the host

> This command will attempt to download the image from the registry if it does not exist on the local host. This is the one command you can run where you can have no local images and it will spin up an active container at the end of its execution. They make it that easy!
{: .prompt-tip }

#### inspect
`docker inspect` enables one to view low-level details about an image or container.

```bash
detrain@detrain:~$ docker inspect nginx
...
```

You may find this command useful to find:
- Environment variables
- Command line options
- Architecture / OS


#### logs
`docker logs` enables one to view log output from a container.

```bash
detrain@detrain:~$ docker logs modest_lederberg 
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/12/22 01:25:07 [notice] 1#1: using the "epoll" event method
2023/12/22 01:25:07 [notice] 1#1: nginx/1.25.3
2023/12/22 01:25:07 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)
...
```

> This command reveals so many details about the application. It will contain runtime successes/failures and proves a useful tool for one to debug issues inside the container.
{: .prompt-tip }

## Summary
This concludes basic usage of docker, the follow on posts will assume command line proficiency (or ability to use `--help`)!
