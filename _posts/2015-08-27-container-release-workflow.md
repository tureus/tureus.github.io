---
layout: post
title:  "Containerization: a workflow to release images"
date:   2015-08-27 14:44:03
categories: devops
---

A pattern I have found in the [heka](https://www.github.com/mozilla-services/heka) project is quite interesting and could be useful to others to understand. How to build optimized images when deploying staticly compiled applications.

The resources required to build an application can be many times more voluminous than what is required to actually run the application. In a debian environment, the developer will need to install all the "-dev" before they can build the project. These "-dev" packages will include source code, headers, and helper executables to put the library to use. You will also need the host language's build environment, which includes binaries and standard libraries which can measure in the many 10s of MBs.

So what can we do to leverage containers for both development and deployment? I'll lay out a scheme that I have found in use on the heka project:

  1. A development environment container, defined in the root of the project as "/Docker"
  2. A build environment container, defined in "/docker/Docker"
  3. A release container, defined in "/docker/Docker.final"

The development environment has a lot of value, it lets contributors get right to modifying source code and running builds. The container specification will list out all the nitty gritty of what packages are required and it will also specify the recommended environment. So if the upstream author is comfortable with Debian they might as well specify "debian:jesse" and install all the required packages.

The build environment takes the development environment and runs the commands necessary to build a production executable. Builds may require additional tools such as debhelper for putting files in to a suitable archive with metadata. In the case of heka, this means building a debian package but build environments could be provided for rpms, atom, snappy, or just plain tar.gz releases. I'm not sure what pattern would be useful for fanning out to these distribution channels -- perhaps a Dockerfile per channel?

The release container is a minimal image which receives a copy of that build. The trick here is the release container is built from inside the build environment. You must map the docker host's /var/run/docker.sock in to the build environment. The build environment copies the release file in to a minimal image which you can then push to your personal registry. Now you have small, purpose built containers which can be deployed just about anywhere!

Pretty cool pattern I didn't know existed until today.
