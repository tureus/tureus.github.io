---
layout: post
title:  "Docker: don't forget to --rm"
date:   2015-08-28 12:05:03
categories: devops
---

Docker is a fascinating tool, it's a mostly-ergonomic wrapper around LXC with a git-like workflow for saving and distributing changes to containers. For a beginner that git-like workflow can hide the artifacts created through everyday usage. Those artifacts use up memory and make debugging more complex (mostly through cluttering command outputs, Docker won't let you get in to a bad state).

The `--rm` flag for `run`
------

Funny thing: I just discovered the `-a` flag on `docker ps`. It showed me all these containers I had left stopped in my environment.

My loaded containers (either running or stopped) looked a little something like this:

    $ docker ps -a
    CONTAINER ID        IMAGE                                                              COMMAND                CREATED             STATUS                      PORTS               NAMES
    29ca95735186        01d1d61243f59a3f8509adc71fd1d215315e6c64a4c7bb3f3901a4b4e1669916   "/bin/sh -c '. ./env   10 minutes ago      Exited (2) 8 minutes ago                        distracted_hoover
    7615bc17affe        f881b53cf19c34aa25905926ceb895066d853a3a1d5809d4004c42008b358c0e   "/bin/sh -c '. env.s   12 minutes ago      Exited (2) 12 minutes ago                       gloomy_yalow

But with 20 more entries in the table. Turns out every `docker run` I had been running was leaving behind these stopped instances.

How did all these images get there? Turns out it's the default behavior. When you execute

    # -it means 'interactive tty'
    docker run -it xrlx/my_image bash
    # do some bash-y stuff in your image's environment
    # exit
    $ docker ps -a
    CONTAINER ID        IMAGE                                                              COMMAND                CREATED             STATUS                      PORTS               NAMES
    e69365353f4f        xrlx/my_image                                                      "bash"                 2 seconds ago       Exited (0) 1 seconds ago                        grave_mestorf

You are leaving the container 'stopped' but still taking up memory. I'm sure the data is paged out on demand, but still, the artifacts are left behind.

The recommended practice is to have one process per container, so add `--rm` to your run command if you know you won't be coming back to that process.

    docker run -it --rm xrlx/my_image bash
    # do some bash-y stuff in your image's environment
    # exit
    $ docker ps -a
    CONTAINER ID        IMAGE                                                              COMMAND                CREATED             STATUS                      PORTS               NAMES
    # EMPTY

This small revelation also solved the mystery of why 'docker rmi <IMAGE_ID>' was saying "image is in use" but `docker ps` was showing no running processes. I had stopped processes I simply couldn't see and didn't know how to clean up.

The `--rm` flag in `build`
-------

How about resource clean up when building images? Each `RUN` stanza in the Dockerfile creates an intermediate image which is linked to the previous image. This is the efficient layering you hear so much about. You may want those intermediate images to enable fast runs of `docker build` but you should realize those images are left behind.

    $ docker images
    REPOSITORY              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    xrlx/heka_base          latest              9ec8276ab451        10 seconds ago      686.9 MB
    xrlx/logstash           0.1.1               15cb2b240e6f        2 days ago          639.7 MB
    xrlx/logstash           latest              15cb2b240e6f        2 days ago          639.7 MB
    xrlx/logstash           0.1.0               6d5bb4d9a810        2 days ago          639.7 MB
    logstash-fully-loaded   latest              5d59a9cc93da        2 days ago          639.7 MB
    mozilla/heka            latest              11a2900cf646        3 days ago          191.6 MB
    mozilla/heka_base       latest              45f94e9c37a3        3 days ago          686.8 MB
    <none>                  <none>              7061e2c8ff3b        4 days ago          868.5 MB
    <none>                  <none>              20668bb0380a        4 days ago          758.4 MB
    logstash                latest              51edd798552c        10 days ago         636.8 MB
    golang                  1.4                 cc4093890da5        10 days ago         517.3 MB
    debian                  jessie              4a5e6db8c069        10 days ago         125.2 MB
    <none>                  <none>              54ea9aa6583e        2 weeks ago         630.6 MB

From this table you can see the intermediate images marked as `<none>`. Their only identfying property is from the `IMAGE ID` column.

When building images and you don't want to keep those intermediate images use the `--rm` flag, whose documenation reads:

    --rm=true             Remove intermediate containers after a successful build

This is a very useful setting. You can test out `docker build` and if the build fails you have the intermediate images for fast runs. Don't edit the `RUN` stanzas of the more constly actions and you will automatically get them cached and ready to use in the future. When you finally get your `docker build` to complete you will benefit from the intermediate images being cleaned up.

Edit: the `--rm` option is set to true by default (as noted in the documentation). I now realize those unnamed images are from failed, abandoned builds.

Conclusion
-----

So there you have it, Docker lets you iterate quickly on images and commands but it could leave considerable amounts of garbage behind. This is some low hanging fruit to keep your Docker host's environment as lean as possible. I would argue the `--rm` should be the default behavior when using `run` but I will keep working with Docker and see if there's a good reason for the default behavior.

ProTip: I wasn't going to start any of those stopped containers again. To clean them up: `docker ps -a | awk '{print $1}' | grep -v CONTAINER | xargs docker rm`