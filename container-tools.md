# Red Hat Container Tools

- [Red Hat Container Tools](#red-hat-container-tools)
  - [An Overview of Container Tools](#an-overview-of-container-tools)
  - [Podman: Familiar Territory](#podman-familiar-territory)
  - [Buildah: Granularity \& Control](#buildah-granularity--control)
    - [Prep Work](#prep-work)
    - [Basic Build](#basic-build)
    - [Using Tools Outside The Container](#using-tools-outside-the-container)
    - [External Build Time Mounts](#external-build-time-mounts)
    - [Cleanup](#cleanup)
    - [Conclusions](#conclusions)
  - [Skopeo: Moving \& Sharing](#skopeo-moving--sharing)
    - [Remotely Inspecting Images](#remotely-inspecting-images)
    - [Pulling Images](#pulling-images)
    - [Moving Between Container Storage (Podman \& Docker)](#moving-between-container-storage-podman--docker)
    - [Moving Between Container Registries](#moving-between-container-registries)
    - [Conclusions](#conclusions-1)
  - [CRIU: Checkpointing and Restoring](#criu-checkpointing-and-restoring)
    - [Conclusions](#conclusions-2)
  - [Udica: Custom SELinux Policies](#udica-custom-selinux-policies)
    - [Conclusions](#conclusions-3)
  - [OSCAP Podman: Trust but Verify](#oscap-podman-trust-but-verify)
    - [Conclusions](#conclusions-4)
  - [Next Up](#next-up)


## An Overview of Container Tools

In this chapter we're going to cover a plethora of container tools available in Red Hat Enterprise Linux (RHEL), including Podman, Buildah, Skopeo, CRIU and Udica. Before we get into the specific tools, it's important to understand how these are tools are provided to the end user in the Red Hat ecosystem.

The RHEL kernel, systemd, and the container tools, centered around [containers](github.com/containers) and [CRI-O](github.com/cri-o/cri-o), serve as the foundation for both **RHEL Server** as well as **RHEL CoreOS**. **RHEL Server** is a flexible, general purpose operating system which can be customized for many different use cases. On the other hand, **RHEL CoreOS** is a minimal, purpose built operating system intended to be consumed within automated environments like OpenShift. This lab will specifically cover the tools available in **RHEL Server**, but much of what you learn is useful with **RHEL CoreOS** which is built from the same bits, but packaged specifically for OpenShift and Edge use cases.

Here's a quick overview of how to think about RHEL Server versus RHEL CoreOS:

1. General Purpose: User -> Podman -> RHEL Server
2. OpenShift: User -> Kubernetes API -> Kubelet -> CRI-O -> RHEL CoreOS

In a RHEL Server environment, the end user will create containers directly on the container host with Podman. In an OpenShift environment, the end user will create containers through the Kubernetes API - users generally do not interact directly with CRI-O on individual hosts in the cluster. **Stated another way, Podman is the primary container interface in RHEL, while Kubernetes is the primary interface in OpenShift**.

For the rest of this lab, we will focus on the container tools provided in RHEL Server. The launch of RHEL8 introduced the concept of [Application Streams](https://www.redhat.com/en/blog/introduction-appstreams-and-modules-red-hat-enterprise-linux), which provide users with access to the latest versions of software like Python, Ruby, and Podman. These Application Streams have different, and often shorter life cycles than RHEL (10+ years). Specifically, RHEL8 Server provides users with two types of Application Streams for Container tools:

1. Fast: Rolling stream which is updated with new versions of Podman and other tools up to every 12 weeks, and only supported until the next version is released. This stream is for users looking for the latest features in Podman.
2. Stable: Traditional streams released once per year, and supported for 24 months. Once released these streams do not update versions of Podman and other tools, and only provide security fixes. This stream is for users looking to put Podman into production depending on stability.

With either stream, the underlying RHEL kernel, systemd, and other packages are treated as a rolling stream. The only choice is is whether to use the fast stream or one of the stable streams. Since RHEL provides a very stable [ABI/API Policy](https://access.redhat.com/articles/rhel8-abi-compatibility) the vast majority of container users will not notice and should not be concerned with kernel, systemd, glibc, etc updates on the container host. If the users selects one of the stable streams, the API to Podman will remains stable and updated for security.

For a deeper dive, check out [RHEL 8 enables containers with the tools of software craftsmanship](https://www.redhat.com/en/blog/rhel-8-enables-containers-tools-software-craftsmanship-0). Now, let's move on to installing and using these different streams of software.

<!-- TODO yum module list doesn't list container-tools -->
<!-- ## Using the Fast and Stable Streams

As mentioned in the previous step, there are two main types of streams. To view them, run the following command:

```
yum module list | grep container-tools
```

Notice there are two types:
1. container-tools:rhel8 - this is the fast moving stream, it's updated once every 12 weeks and generally fixes bugs by rolling to new versions
2. container-tools:1.0 - this was released with RHEL 8.0 and supported for 24 months, and receives bug fixes with back ports that keep the API and CLI interfaces stable
3. container-tools:2.0 - this was released with RHEL 8.2 and supported for 24 months, and receives bug fixes with back ports that keep the API and CLI interfaces stable

Now, let's pretend we are developer looking for access to the latest features in RHEL. Let's inspect the description of the fast moving stream. Notice that there are multiple versions of the rhel8 application stream. Every time a package is updated the entire group of packages is version controlled and tested together:

```
yum module info container-tools:rhel8
```

Now, let's install the fast moving container-tools:rhel8 Application Stream like this:

```
yum module install -y container-tools:rhel8
```

We should have a whole set of tools installed:

```
yum module list --installed
```

Look at the packages that were installed as part of this Application Stream:

```
yum module repoquery --installed container-tools
```

Look at the version of Podman that was installed. It should be fairly new, probably within a few months of what's latest upstream:

```
podman -v
```

Let's clean up the environment, and start from scratch:

```
yum module remove -y container-tools
yum module reset -y container-tools
```

OK, now let's pretend we are a systems administrator or SRE that wants a set of stable tools which are supported for 24 months. First, inspect the stable stream that was released in RHEL 8.0. Notice that there are several versions of this Application Stream. Every time a package is updated a new stream version is generated to snapshot the exact versions of each package together as a stream:

```
yum module info container-tools:1.0
```

Now, install it:

```
yum module install -y --allowerasing container-tools:1.0
```

Check the version of Podman again:

```
podman -v
```

Notice that it's an older version of Podman. This version only gets back ports and will never move beyond Podman 1.0.2. Note, there is no connection between the container-tools version number and the Podman version number. It is purely coincidence that these numbers coincide. The container-tools version number is an arbitrary number representing all of the tools tested together in the Application Stream. This includes, Podman, Buildah, Skopeo, CRIU, etc.

Now, let's go back to the latest version of the container-tools for the rest of this module:

```
yum module remove -y container-tools
yum module reset -y container-tools
yum module install -y container-tools:rhel8
```

Notice how easy it was to move between the stable streams and the fast moving stream. This is the power of modularity. Now, let's move on to using the actual tools. -->

## Podman: Familiar Territory

The goal of this lab is to introduce you to Podman and some of the features that make it interesting. If you have ever used Docker, the basics should be pretty familiar. Lets start with some simple commands.

Pull an image:

```bash
podman pull ubi8
```

List locally cached images:

```bash
podman images
```

Start a container and run bash interactively in the local terminal. When ready, exit:

```bash
podman run -it ubi8 bash
```

```bash
exit
```

List running containers:

```bash
podman ps -a
```

Now, let's move on to some features that differentiates Podman from Docker. Specifically, let's cover the two most popular reasons - Podman runs without a daemon (daemonless) and without root (rootless). Podman is an interactive command more like bash, and like bash it can be run as a regular user (aka. rootless).

The container host we are working with already has a user called RHEL, so let's switch over to it. Note, we set a couple of environment variables manually because we're using the switch user command. These would normally be set at login:

```
su - rhel
export XDG_RUNTIME_DIR=/home/rhel
``` -->

Now, fire up a simple container in the background:

```bash
podman run -id ubi8 bash
```

Now, lets analyze a couple of interesting things that makes Podman different than Docker - it doesn't use a client server model, which is useful for wiring it into CI/CD systems, and other schedulers like Yarn:

Inspect the process tree on the system:

```bash
pstree -Slnc
```

You should see something similar to:

```bash
└─conmon─┬─{conmon}
         └─bash(ipc,mnt,net,pid,uts)
```

There's no Podman process, which might be confusing. Lets explain this a bit. What many people don't know is that containers disconnect from Podman after they are started. Podman keeps track of metadata in `~/.local/share/containers` (`/var/lib/containers` is only used for containers started by root) which tracks which containers are created, running, and stopped (killed). The metadata that Podman tracks is what enables a `podman ps` command to work.

In the case of Podman, containers disconnect from their parent processes so that they don't die when Podman exits. In the case of Docker and CRI-O which are daemons, containers disconnect from the parent process so that they don't die when the daemon is restarted. For Podman and CRI-O, there is utility which runs before runc called conmon (Container Monitor). The conmon utility disconnects the container from the engine by doing forking twice (called a double fork). That means, the execution chain looks something like this with Podman:

`bash -> podman -> conmon -> conmon -> runc -> bash`

Or like this with CRI-O:

`systemd -> crio -> conmon -> conmon -> runc -> bash`

Or like this with Docker engine:

`systemd -> dockerd -> containerd -> docker-shim -> runc -> bash`

Conmon is a very small C program that monitors the standard in, standard error, and standard out of the containerized process. The conmon utility and docker-shim both serve the same purpose. When the first conmon finishes calling the second, it exits. This disconnects the second conmon and all of its child processes from the container engine. The second conmon then inherits init system (systemd) as its new parent process. This daemonless and simplified model which Podman uses can be quite useful when wiring it into other larger systems, like CI/CD, scripts, etc.

Podman doesn't require a daemon and it doesn't require root. These two features really set Podman apart from Docker. Even when you use the Docker CLI as a user, it connects to a daemon running as root, so the user always has the ability escalate a process to root and do whatever they want on the system. Worse, it bypasses sudo rules so it's not easy to track down who did it.

Now, let's move on to some other really interesting features. Rootless containers use a kernel feature called User Namespaces. This maps the one or more user IDs in the container to one or more user IDs outside of the container. This includes the root user ID in the container as well as any others which might be used by programs like nginx or Apache.

Podman makes it super easy to see this mapping. Start an nginx container to see the user and group mapping in action:

```bash
podman run -id registry.access.redhat.com/rhscl/nginx-114-rhel7 nginx -g 'daemon off;'
```

Now, execute the Podman bash command:

```bash
podman top -l args huser hgroup hpid user group pid seccomp label
```

Notice that the host user, group and process ID *in* the container all map to different and real IDs on the host system. The container thinks that nginx is running as the user `default` and the group `root` but really it's running as an arbitrary user and group. This user and group are selected from a range configured for the `student` user account on this system. This list can easily be inspected with the following commands:

```shell
cat /etc/subuid
```

You will see something similar to this:

```shell
student:165536:65536
```

The first number represents the starting user ID, and the second number represents the number of user IDs which can be used from the starting number. So, in this example, our `student` user can use 65,535 user IDs starting with user ID 165536. The Podman bash command should show you that nginx is running in this range of UIDs.

The user ID mappings on your system might be different because shadow utilities (useradd, usderdel, usermod, groupadd, etc) automatically creates these mappings when a user is added. As a side note, if you've updated from an older version of RHEL, you might need to add entries to /etc/subuid and /etc/subgid manually.

OK, now stop all of the running containers. No more one liners like with Docker, it's just built in with Podman:

```bash
podman kill --all
```

Remove all of the actively defined containers. It should be noted that this might be described as deleting the copy-on-write layer, config.json (commonly referred to as the Config Bundle) as well as any state data (whether the container is defined, running, etc):

```bash
podman rm --all
```

We can even delete all of the locally cached images with a single command:

```bash
podman rmi --all
```

The above commands show how easy and elegant Podman is to use. Podman is like a Chef's knife. It can be used for pretty much anything that you used Docker for, but let's move on to Builah and show some advanced use cases when building container images.

## Buildah: Granularity & Control

The goal of this lab is to introduce you to Buildah and the flexibility it provides when you need to build container images your way. There are a lot of different use cases that just "feel natural" when building container images, but you often, you can't quite wire together and elegant solutions with the client server model of existing container engines. In comes Buildah. To get started, lets introduce some basic decisions you need to think through when building a new container image.

1. Image vs. Scratch: Do you want to start with an existing container image as the source for your new container image, or would you prefer to build completely from scratch? Source images are the most common route, but it can be nice to build from scratch if you have small, statically linked binaries.

2. Inside vs. Outside: Do you want to execute the commands to build the next container image layer inside the container, or would you prefer to use the tools on the host to build the image? This is completely new concept with Buildah, but with existing container engines, you always build from within the container. Building outside the container image can be useful when you want to build a smaller container image, or an image that will always be ran read only, and never built upon. Things like Java would normally be built in the container because they typically need a JVM running, but installing RPMs might happen from outside because you don't want the RPM database in the container.

3. External vs. Internal Data: Do you have everything you need to build the image from within the image? Or, do you need to access cached data outside of the build process? For example, It might be convenient to mount a large cached RPM cache inside the container during build, but you would never want to carry that around in the production image. The use cases for build time mounts range from SSH keys to Java build artifacts - for more ideas, see this [GitHub issue](https://github.com/moby/moby/issues/14080).

Alright, let's walk through some common scenarios with Buildah.

### Prep Work
Just like Podman, Buildah can execute in rootless mode, but since you have tools on the container host interacting files in the container image, you need to make Buildah think it's running as root. Buildah comes with a cool sub-command called unshare which does just this. It puts our shell into a user namespace just like when you have a root shell in a container. The difference is, this shell has access to tools installed on the container host, instead of in the container image. Before we complete the rest of this lab, execute the "buildah unshare" command. Think of this as making yourself root, without actually making yourself root:

```
buildah unshare
```

Now, look at who your shell thinks you are:

```
whoami
```

It's looks like you are root, but you really aren't, but let's prove it:

```
touch /etc/shadow
```

The touch command fails because you're not actually root. Really, the touch command executed as an arbitrary user ID in your /etc/subuid range. Let that sink in. Linux containers are mind bending. OK, let's do something useful.

### Basic Build

First declare what image you want to start with as a source. In this case, we will start with Red Hat Universal Base Image:

```
buildah from ubi8
```

This will create a "reference" to what Buildah calls a "working container" - think of them as a starting point to attach mounts and commands. Check it out here:

```
buildah containers
```

Now, we can mount the image source. In effect, this will trigger the graph driver to do its magic, pull the image layers together, add a working copy-on-write layer, and mount it so that we can access it just like any directory on the system:

```
buildah mount ubi8-working-container
```

Now, lets add a single file to the new container image layer. The Buildah mount command can be ran again to get access to the right directory:

```
echo "hello world" > $(buildah mount ubi8-working-container)/etc/hello.conf
```

Lets analyze what we just did. It's super simple, but kind of mind bending if you come from using other container engines. First, list the directory in the copy-on-write layer:

```
ls -alh $(buildah mount ubi8-working-container)/etc/
```

You should see hello.conf right there. Now, cat the file:

```
cat $(buildah mount ubi8-working-container)/etc/hello.conf
```

You should see the text you expect. Now, lets commit this copy-on-write layer as a new image layer:

```
buildah commit ubi8-working-container ubi8-hello
```

Now, we can see the new image layer in our local cache. We can view it with either Podman or Buildah (or CRI-O for that matter, they all use the same image store):

```
buildah images
```

```
podman images
```

When we are done, we can clean up our environment quite nicely. The following command will delete references to "working containers" and completely remove their mounts:

```
buildah delete -a
```

But, we still have the new image layer just how we want it. This could be pushed to a registry server to be shared with others if we like:

```
buildah images
```

```
podman images
```

### Using Tools Outside The Container

Create a new working container, mount the image, and get a working copy-on-write layer:

```
WORKING_MOUNT=$(buildah mount $(buildah from scratch))
echo $WORKING_MOUNT
```

Verify that there is nothing in the directory:

```
ls -alh $WORKING_MOUNT
```

Now, lets install some basic tools:

```
yum install --installroot $WORKING_MOUNT bash coreutils --releasever 8 --setopt install_weak_deps=false -y
yum clean all -y --installroot $WORKING_MOUNT --releasever 8
```

Verify that some files have been added:

```
ls -alh $WORKING_MOUNT
```

Now, commit the copy-on-write layer as a new container image layer:

```
buildah commit working-container minimal
```

Now, test the new image layer, by creating a container:

```
podman run -it minimal bash
```

```
exit
```

Clean things up for our next experiment:

```
buildah delete -a
```

We have just created a container image layer from scratch without ever installing RPM or YUM. This same pattern can be used to solve countless problems. Makefiles often have the option of specifying the output directory, etc. This can be used to build a C program without ever installing the C toolchain in a container image layer. This is best for production security where we don't want the build tools laying around in the container.

### External Build Time Mounts

As a final example, lets use a build time mount to show how we can pull data in. This will represent some sort of cached data that we are using outside of the container. This could be a repository of Ansible Playbooks, or even Database test data:

```
mkdir ~/data
dd if=/dev/zero of=~/data/test.bin bs=1MB count=100
ls -alh ~/data/test.bin
```

Now, lets fire up a working container:

```
buildah from ubi8
buildah mount ubi8-working-container
```

To consume the data within the container, we use the buildah-run subcommand. Notice that it takes the -v option just like "run" in Podman. We also use the Z option to relabel the data for SELinux. The dd command simply represents consuming some smaller portion of the data during the build process:

```
buildah run -v ~/data:/data:Z ubi8-working-container dd if=/data/test.bin of=/etc/small-test.bin bs=100 count=2
```

Commit the new image layer and clean things up:

```
buildah commit ubi8-working-container ubi8-data
buildah delete -a
```

Test it and note that we only kept the pieces of the data that we wanted. This is just an example, but imagine using this with a Makefile cache, or Ansible playbooks, or even a copy of production database data which needs to be used to test the image build or do a schema upgrade, which must be accessed during the image build process. There are tons of places where you need to access data, only at build time, but don't want it during production deployment:

```
podman run -it ubi8-data ls -alh /etc/small-test.bin
```

### Cleanup

Exit the user namespace:

```
exit
```

### Conclusions

Now, you have a pretty good understanding of the cases where Buildah really shines. You can start from scratch, or use an existing image, use tools installed on the container host (not in the container image), and move data around as needed. This is a very flexible tool that should fit quite nicely in your tool belt. Buildah lets you script builds with any language you want, and build tiny images with only the bare minimum of utilities needed inside the image.

Now, lets move on to sharing containers with Skopeo...

## Skopeo: Moving & Sharing

In this step, we are going to do a couple of simple exercises with Skopeo to give you a feel for what it can do. Skopeo doesn't need to interact with the local container storage (.local/share/containers), it can move directly between registries, between container engine storage, or even directories.

### Remotely Inspecting Images

First, lets start with the use case that kicked off the Skopeo project. Sometimes, it's really convenient to inspect an image remotely before pulling it down to the local cache. This allows us to inspect the metadata of the image and see if we really want to use it, without synchronizing it to the local image cache:

```
skopeo inspect docker://registry.fedoraproject.org/fedora
```

We can easily see the "Architecture" and "Os" metadata which tells us a lot about the image. We can also see the labels, which are consumed by most container engines, and passed to the runtime to be constructed as environment variables. By comparison, here's how to see this metadata in a running container:

```
podman run --name metadata-container -id registry.fedoraproject.org/fedora bash
podman inspect metadata-container
```

### Pulling Images

Like, Podman, Skopeo can be used to pull images down into the local container storage:

```
skopeo copy docker://registry.fedoraproject.org/fedora containers-storage:fedora
```

But, it can also be used to pull them into a local directory:

```
skopeo copy docker://registry.fedoraproject.org/fedora dir:$HOME/fedora-skopeo
```

This has the advantage of not being mapped into our container storage. This can be convenient for security analysis:

```
ls -alh ~/fedora-skopeo
```

The Config and Image Layers are there, but remember we need to rely on a [Graph Driver](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/#h.kvykojph407z) in a [Container Engine](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/#h.6yt1ex5wfo3l) to map them into a RootFS.

<!-- 
### Moving Between Container Storage (Podman & Docker)

First, let's do a little hack to install Docker CE side by side with Podman on RHEL 8. Don't do this on a production system as this will overwrite the version of runc provided by Red Hat:

```
yes|sudo rpm -ivh --nodeps --force https://download.docker.com/linux/centos/8/x86_64/stable/Packages/containerd.io-1.3.7-3.1.el8.x86_64.rpm
sudo yum install -y docker-ce
```

Now, enable the Docker CE service:

```
sudo systemctl enable --now docker
```

Now that we have Docker and Podman installed side by side with the Docker daemon running, lets copy an image from Podman to Docker. Since we have the image stored locally in .local/share/containers, it's trivial to copy it to /var/lib/docker using the daemon:

```
skopeo copy containers-storage:registry.fedoraproject.org/fedora docker-daemon:registry.fedoraproject.org/fedora:latest
```

Verify that the repository is now in the Docker CE cache:

```
docker images | grep registry.fedoraproject.org
```

This can be useful when testing and getting comfortable with other OCI complaint tools like Podman, Buildah, and Skopeo. Sometimes, you aren't quite ready to let go of what you know so having them side by side can be useful. Remember though, this isn't supported because it replaces the runc provided by Red Hat.
--->

### Moving Between Container Registries

Finally, lets copy from one registry to another. I have set up a writeable repository under my username (fatherlinux) on quay.io. To do this, you have to use the credentials provided below. Notice, that we use the "--dest-creds" option to authenticate. We can also use the "--source-cred" option to pull from a registry which requires authentication. This tool is very flexible. Designed by engineers, for engineers.

```
skopeo copy docker://registry.fedoraproject.org/fedora docker://quay.io/fatherlinux/fedora --dest-creds fatherlinux+fedora:5R4YX2LHHVB682OX232TMFSBGFT350IV70SBLDKU46LAFIY6HEGN4OYGJ2SCD4HI
```

This command just synchronized the fedora repository from the Fedora Registry to Quay.io without ever caching it in the local container storage. Very cool right?

Finally, exit the ''rhel'' user because we need root for the next lab:

```
exit
```

### Conclusions

You have a new tool in your tool belt for sharing and moving containers. Hopefully, you find other uses for Skopeo.

<!-- 
## CRIU: Checkpointing and Restoring

With some help from a program called CRIU, Podman can checkpoint and restore containers on the same host. This can be useful with workloads that have a long startup period or require a long time to warm up caches. For example, large memcached servers, database, or even Java workloads can take several minutes or even hours to reach maximum throughput performance. This is often referred to as cache warming.

If this doesn't quite make sense, let's talk about it in the context of container creation and deletion. Podman allow you to break the creation and deletion of containers down into very granular steps. Here's what the life cycle of a container looks like from start to finish:

1. podman pull - Pull the container image
2. podman create - Add tracking metadata to /var/lib/containers or .local/share/containers
3. podman mount - Create a copy-on-write layer and mount the container image with a read/write layer above it
4. podman init - Create a config.json file
5. podman start - Run the workload by handing the config.json and root file system to runc
6. Workload runs either as a batch process, or as a daemon
7. podman kill - kills the process or processes in the container
8. podman rm - Unmount and delete the copy-on-write layer
9. podman rmi - remove the image /var/lib/containers or .local/share/containers

To understand CRIU, you need to understand step 6. When this step is executed, Podman sends a kill signal to the processes in the container. CRIU allows us to break this down even further like this:


1. podman pull - Pull the container image
2. podman create - Add tracking metadata to /var/lib/containers or .local/share/containers
3. podman mount - Create a copy-on-write layer and mount the container image with a read/write layer above it
4. podman init - Create a config.json file
5. podman start - Run the workload by handing the config.json and root file system to runc
6. Workload runs either as a batch process, or as a daemon
7. podman checkpoint - Dump contents of memory to disk and kill processes
8. Workload process no longer running, memory contents are saved on disk
9. podman restore - Restore memory contents to new processes
10. Workload runs either as a batch process, or as a daemon
11. podman kill - kills the process or processes in the container
12. podman rm - Unmount and delete the copy-on-write layer
13. podman rmi - remove the image /var/lib/containers or .local/share/containers

So, in a nutshell, CRIU gives you more flexibility with containerized processes. Let's see it in action. First, start a simple container which generates incrementing numbers so that we can verify memory contents are really restored:

```
podman run -d --name looper ubi8 /bin/sh -c \
         'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
```

Now, verify that numbers are being generated. Run this a few times to see the numbers incrementing:

```
podman logs -l
```

Now, let's dump the contents of memory to disk, and kill the process:

```
podman container checkpoint -l
```

Verify that it's not running. Notice that that container is in the exited state. This means the copy-on-write layer for the container has not been deleted. Since we used the checkpoint sub-command, the contents of memory are also saved on disk:

```
podman ps -a
```

Verify that numbers are not being generated. Run this a few times to verify:

```
podman logs -l
```

Restore the container:

```
podman container restore -l
```

Verify the contents of memory and disk are being used and the numbers are incrementing again:

```
podman logs -l
```

We're all done, so clean up. This will kill the process, delete the contents of the copy-on-write layer, and remove all of the metadata for all containers:

```
podman kill -a
```

### Conclusions

Checkpointing and restoring containers is easy with CRIU and Podman. As part of the container-tools application streams, specific versions of Podman and CRIU are tested and verified to work together (not all versions of Podman and CRIU are guaranteed to work together). Now, let's move on to some more tools.

## Udica: Custom SELinux Policies

The default Containers SELinux policy does a really good job Podman in RHEL 8. Most containers "just work" but like any security tool, every now and then we need to make some customizations. Sometimes, especially as we expand the types of workloads that we run in containers, we bump into places where SELinux blocks us. Udica allows an administrator to customize the SELinux policy specifically for a workload for without being an SELinux expert.

For example, run the following container:

```
podman run --name home-test -v /home/:/home:ro -it ubi8 ls /home
```

The above command will fail because the default SELinux policy does not allow containers to mount /home as read only. We can verify that there are no allow rules which permit this command to be executed:

```
sesearch -A -s container_t -t home_root_t -c dir -p read
```

With Udica, we can quickly and easily configure SELinxux to allow us to mount /home as root. First, we have to extract the metadata from our container:

```
podman inspect home-test > home-test.json
```

Now, Udica will analyze this data and create a custom SELinux policy for us:

```
udica -j home-test.json home_test
```

Use the SELinux tools to load the new policy:

```
semodule -i home_test.cil /usr/share/udica/templates/{base_container.cil,home_container.cil}
```

Now, run the same type of container again, but pass it a security option telling it to label the process to use our new custom policy, and it will execute without being blocked. First start the contaier:

```
podman run --name home-test-2 --security-opt label=type:home_test.process -v /home/:/home:ro -id ubi8 bash
```

Execute the ''ls'' command:

```
podman exec -it home-test-2 ls /home
```

You will notice that the process is running with the "home_test.process" SELinux context:

```
ps -efZ | grep home_test
```

We can also verify that there is a new rule in this policy to allow our container to mount /home read only:

```
sesearch -A -s home_test.process -t home_root_t -c dir -p read
```

-->
### Conclusions

It's always best to have SELinux enabled, especially with containers. It's so easy to create a custom SELinux policy with Udica, that you should never disable it. If you'd like to understand Udica a bit deeper, check out this great article, [Use udica to build SELinux policy for containers](https://fedoramagazine.org/use-udica-to-build-selinux-policy-for-containers/) by Lukas Vrabec. Now, let's move on to another tool.

## OSCAP Podman: Trust but Verify

In this lab we're going to demonstrate using the [oscap-podman](https://github.com/OpenSCAP/openscap/blob/master/utils/oscap-podman) command to verify the security of the [Red Hat Universal Base Image](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image). Red Hat UBI is a rock solid, freely distributable container base image created by Red Hat and build on RHEL bits (see also: [Comparison of Container Images](http://crunchtools.com/comparison-linux-container-images/). UBI is rebuilt every six weeks or every time there is a Critical or Important [CVE](https://www.redhat.com/en/topics/security/what-is-cve)(Common Vulnerabilities and Exposures) patched and is the foundation for all containerized products at Red Hat. See also [UBI FAQ](https://developers.redhat.com/articles/ubi-faq).

While UBI is a great foundation, some of us are skeptical and like our vendors to show their work. Luckily, the Red Hat Product Security Team does just that using OVAL data. For those that have never heard of this, OVAL stands for The Open Vulnerability and Assessment Language project, which maintained by The MITRE Corporation (same group that does CVEs), an international, information security effort that promotes open and publicly available security content, and seeks to standardize the transfer of this information across the entire spectrum of security tools and services. Red Hat Product Security helps customers evaluate and manage risk by tracking and investigating all security issues affecting Red Hat customers and providing timely and concise patches and security advisories via the Red Hat Customer Portal creating and supporting OVAL patch definitions, providing a machine-readable versions of our security advisories. This allows OVAL-compatible tools to test for the presence of vulnerabilities for which Red Hat has released security updates that could be applied to the system.

You can see that data here: https://www.redhat.com/security/data/oval/v2/RHEL8/

Before we begin the lab, download the latest OVAL data from Red Hat Product Security:

```
wget https://www.redhat.com/security/data/oval/v2/RHEL8/rhel-8.oval.xml.bz2
```

Now, we're going to use the oscap-podman tool to consume the data and compare some versions of the Red Hat Universal Base Image which we have stored locally in the Podman cache. Container images age like cheese, not like wine. Older container images which haven't been rebuilt will build up CVEs over time. From the Red Hat Ecosystem Catalog, we can see that the [UBI 8.0-126 image](https://catalog.redhat.com/software/containers/ubi8/ubi/5c359854d70cc534b3a3784e?tag=8.0-126&push_date=1560882955000) which was build June 11th, 2019 has a grade of F and at the time of this writing had 3 important security vulnerabilities.

Check it out here: [UBI 8.0-126 image](https://catalog.redhat.com/software/containers/ubi8/ubi/5c359854d70cc534b3a3784e?tag=8.0-126&push_date=1560882955000)

Now, let's take a more detailed look, using the oscap-podman command:

```
oscap-podman registry.access.redhat.com/ubi8/ubi:8.0-126 oval eval --report ./assets/html/ubi-8.0-126-report.html rhel-8.oval.xml.bz2
```

This will create a report. We've already set up a web server (in a container) so that you can quickly look at the  *OSCAP Report 8.0-126* Tab.

Notice that there are many orange lines displaying the CVEs/RHSAs. If you built and ran an application on this very old container image, it would be exposed to these vulnerabilities. Now, let's scan the latest version of UBI provided by Red Hat:

```
oscap-podman registry.access.redhat.com/ubi8/ubi:latest oval eval --report ./assets/html/ubi-latest-report.html rhel-8.oval.xml.bz2
```

Look at the new report from thr *OSCAP Report latest* Tab.

### Conclusions

In general you should almost never see any unapplied patches in this latest build of UBI. If you do, it's like a small window between the time that Red Hat Product Security has issued a CVE, and our product team has rebuilt and published the UBI image. The oscap-podman command allows you to verify this and is useful in "trust but verify" operations like CI/CD system tests. For more information and deeper dive on OSCAP-Podman, check out this great video by Brian Smith: [Scanning Containers for Vulnerabilities on RHEL 8.2 With OpenSCAP and Podman](https://www.youtube.com/watch?v=nQmIcK1vvYc)

## Next Up

Congratulations! You just completed the second module of today's workshop. Please continue to [Developing on OpenShift](./developing-on-openshift-getting-started.md).