---
title: "my receipe for writing dockerfile"
date: "2022-04-20"
tags:
- docker
- container
---

A beginner's guide to write acceptable dockerfiles
<!--more-->

## ov
- use alpine linux or tag with "slim"
  - smaller 
- use multi-stage build
- combine multiple RUNs in one RUN
  - logically the commands are in a group
- use `.dockerignore`
- use `dive`, a tool to check how to slim down image size
  - [dive](https://github.com/wagoodman/dive)
- use distroless by google
  - [github](https://github.com/GoogleContainerTools/distroless)

## caching
- to better utilize caching of the intermediate images, move the parts that are unlikely to change at the top of the dockerfile 
  - like `ADD`, `COPY`
  - requirements install

## inspect
- view the size of intermediate images
  - `docker image history <image>:<tag>`
- dive

## apt-related
- combine `apt-get update` and `apt-get install`, and append `rm -rf /var/lib/apt/lists/*` in the end
  - state information for each package resource specified in apt's `sources.list`
  - see [this](https://askubuntu.com/questions/179955/var-lib-apt-lists-is-huge)

## gosu
- when you need to use `sudo`, use `gosu` instead
  - [why to use gosu?](https://zhuanlan.zhihu.com/p/151915585)
- commonly, `gosu nobody true` is used to confirm that the installed gosu is OK
- typically gosu is later used in `entrypoint.sh`
- [how to install gosu in dockerfile?](https://github.com/tianon/gosu/blob/master/INSTALL.md)
- commonly used to step from root to non-root user and run the app
  - e.g. `exec gosu <username> <command>` will run the command as username

## ref
- [best practice for dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [highly recommended blog](https://towardsdatascience.com/slimming-down-your-docker-images-275f0ca9337e)
- [blog: distroless](https://learnk8s.io/blog/smaller-docker-images)
- [zhihu: scratch or alpine+](https://zhuanlan.zhihu.com/p/115845957)
- [elasticsearch dockfile analysis](https://www.kancloud.cn/hanxt/elk/158426)
- [good blog summarizinig tips for writing dockerfiles](https://jkutner.github.io/2021/04/26/write-good-dockerfile.html)

