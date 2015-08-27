---
layout: post
title:  "Containerization: a workflow to release images"
date:   2015-08-27 14:44:03
categories: devops
---

A pattern I have found in the [heka](https://www.github.com/mozilla-services/heka) project is quite interesting and could be useful to others to understand. How to build optimized images when deploying staticly compiled applications.

The resources required to build an application can be many times more voluminous than what is required to actually run the application. In a debian environment, the developer will need to install all the "-dev" before they can build the project. These "-dev" packages will include source code, headers, and helper executables to put the library to use. You will also need the host language's build environment, which includes binaries and standard libraries which can measure in the many 10s of MBs.

So what can we do to leverage containers for both development and deployment? I'll lay out a scheme that I have found in use on the heka project:

  1) A development environment container, defined in the root of the project as "/Docker"
  2) A build environment container, defined in "/docker/Docker"
  3) A release container, defined in "/docker/Docker.final"

The mechanics of 