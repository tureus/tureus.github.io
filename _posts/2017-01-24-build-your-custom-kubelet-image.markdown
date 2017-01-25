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

History of this Openstack Work
---

The history of the V2 API integrations are confusing at best:

  * November 7th, 2016: `anguslees` adds latest openstack library gophercloud but keeps outdated rackspace-flavored gophercloud. Kubernetes includes two copies of the library. PR unmerged as of today: https://github.com/kubernetes/kubernetes/pull/36344
  * November 16th, 2016: `NickrenREN` swaps out V1 support for V2, has no notion of November 7th work: https://github.com/kubernetes/kubernetes/pull/36976
  * December 6th, 2016: maintainers request V1 backwards compat. Work is stopped by NickrenREN.
  * January 7th, 2017: `mkutsevol` opens issue referencing `anguslees` work, asking if there is interest in rebasing and continuing: https://github.com/kubernetes/kubernetes/issues/39572
  * January 12th, 2017: `mkustevol` starts a work-in-progress branch on their fork. The branch cherry pick `anguslees` gophercloud update, another openstack-related PR, and contributes original work for V1/V2 backwards compat: https://github.com/mkutsevol/kubernetes/tree/feature/openstack_cinder_v1_2_auto
  * January 22nd, 2017: I, `xrl`, work through some unit tests and push the V2 support over the finish line. I have not tackled the version discovery code, in my configuration I specify V2.

I tried unsuccessfully to merge `anguslees` work in to the `kubernetes-1.5`-stable branch. I try again with `mkutsevol`s branch against Kubernetes `master`, working through the merge conflicts related to `godeps`. I do not know how work is merged in to kubernetes tags, leaving me a little too close to the bleeding edge. But let's move on to how I built and shipped my work.

My changes are here: https://github.com/tureus/kubernetes/tree/openstack-v2

Setting up the Kubenetes Development Environment
---

You will need to understand Go in order to do development on Kubernetes. And you should understand how to set up the `$GOPATH`. I highly recommend you create a go environment for exclusive Kubernetes use -- it'll make `godep` diffs easier to manage.

```
mkdir -p $HOME/code/kubernetes-workspace
export GOPATH=$HOME/code/kubernetes-workspace
mkdir -p $GOPATH/src/k8s.io
cd $GOPATH/src/k8s.io
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
```

It's weird how they wanted the `k8s.io` TLD but they hold the code on github -- it's little off the beaten path and another thing for newcomers to understand.


Building That Custom Kubernetes Release
---

Kubernetes, as you may know, has many moving parts. The big one for my OpenStack upgrade is the `kubelet` -- the process which runs on each kubernetes node/master and handles all the necessary local changes. the `kubelet` manages docker container lifecycles (created, update, destroy, etc) and manages cloud API calls, such as creating a cinder volume. I needed to build and dockerize my kubelet using my custom work.

Remember, the `kubelet` code is now built in to a mulipurpose `hyperkube` binary. Previous version of kubernetes used distinct binaries for master/minion/apiserver but now `hyperkube` is the multipurpose executable of choice.

First big task, build the cross compile `hyperkube` for Linux from my OS X machine. Good thing Go makes cross compiling so easy and it's already built in to my tool chain. Just gotta find the right folder/make file/incantation:

```
# you may need to use brew to install a newer bash, as was the case for me
kubernetes$ bash build/run.sh make cross KUBE_FASTBUILD=true ARCH=amd64
```

Without the `KUBE_FASTBUILD` you will cross compile a matrix of platforms and waste 45mins of your time. Should take 10-15 minutes when compiling for linux/amd64. Note that `KUBE_FASTBUILD` is nowhere in the docs, I just trawled the `Makefile` scripts until hitting pay dirt.

The second big task is to put the cross compiled binary in to a docker image:

```
cd cluster/images/hyperkube/
hyperkube$ make VERSION=$YOURCOOLTAG ARCH=amd64
# it downloads some base images, and creates your release image
hyperkube$ docker images
REPOSITORY                                   TAG                           IMAGE ID            CREATED             SIZE
gcr.io/google_containers/hyperkube-amd64     $YOURCOOLTAG                  61fff5278a05        About an hour ago   577 MB
```

As you can see, this script is used by the kubernetes release machinery to push out builds for each release tag. This leaves me satisfied because that's exactly what I wanted to replace with my custom build.

I retag and push to my repo:

```
docker tag gcr.io/google_containers/hyperkube-amd64:$YOURCOOLTAG registry.xrl/hyperkube-amd64:$YOURCOOLTAG
docker push registry.xrl/hyperkube-amd64:$YOURCOOLTAG
```

now the image is in a globally accessible place. How you deploy this custom image will vary greatly depending on your kubernetes deployment stack. I am a happy user of Kargo, the ansible-powered Kubernetes cluster manager: https://github.com/kubernetes-incubator/kargo .

Kargo has a specific call out for Kubernetes images:

```
$ cat roles/download/defaults/main.yml
# SNIP
hyperkube_image_repo: "registry.xrl/hyperkube-amd64:$YOURCOOLTAG"
hyperkube_image_tag: "master-custom"
# SNIP
```

I can then run ansible as usual and kargo will swap out the images:

```
ansible-playbook -i inventory/inventory cluster.yml
```

Conclusion
---

Hope these build notes help you out. Go is a very hackable language with lots of great libraries. And Kubernetes is very active project, pulled in many directions, with a lot of great hackers working on it. But sometimes you have to roll up your sleeves.

Further Reading
---

The documentation is lacking for this specific hackery, but here are some bits that got me going:

 * https://github.com/kubernetes/community/blob/master/contributors/devel/godep.md
 * https://github.com/kubernetes/kubernetes/blob/master/cluster/images/hyperkube/README.md
