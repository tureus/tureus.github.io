---
layout: post
title:  "Kubernetes: Build Your Custom Kubelet Image"
date:   2017-01-24 14:44:03
categories: devops
---

I have been working through OpenStack V2 stability changes for Kubernetes. I want to get the benefits of Kubernetes' container scheduler and the benefits of my large quota on a private OpenStack installation. The big trick, my OpenStack cluster will be upgrading out of the V1 block storage API and only V2 will be available, cutting me off from mainline Kubernetes!

Good thing more experienced OpenStack hackers have worked on the intration. The bad news is that their work is not merged in to master! This post is all about how I grabbed their mildly bitrot code and then built and shipped it for my own testing.

Getting Godep
---

I am not very familiar with the `godep` tool, preferring to vendor all my dependencies manually. But in a big (and I mean BIG) project like Kubernetes it's good to add some method to the madness of managing the many dependencies. So as part of this  build exercise I had to get the `godep` tool.

```
#
export GOBIN=$HOME/.local/bin
go get github.com/tools/godep
go install github.com/tools/godep
```

Which you'd think would work, but there are bugs in `v77` (today's release) which make it incompatible with Kubernetes HEAD. So we need an older version!

```
cd $GOPATH/github.com/tool/godep
git checkout v74
go install github.com/tools/godep
```

Getting The Changes
---

The V2 API upgrade was done by the fine contributor Angus Lees but never merged. You can see the PR here: 
