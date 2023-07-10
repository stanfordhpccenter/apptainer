<- [previous](/00-introduction/README.md) - [home](/README.md) - [next](/02-the-ecosystem/README.md) ->

---
# Downloading and Interacting with Containers

### Objectives

- understand how to get help from the Apptainer documentation 
- learn what it means to `pull` a container and how to do so
- learn how to use the Apptainer "action" commands (`shell`, `run`, `exec`)
- use the `inspect` command to get more info out of containers

This section will be useful for becoming comfortable with using containers. In the coming sections we'll explore topics more geared toward building your own containers and pushing them to registries etc. But it will be useful to first understand _what you want to build_ and interacting with some containers will give you a basis for this. 

## The Built-in Apptainer help

Like a lot of large programs, Apptainer has extensive internal built-in documentation. You can use this to explore new topics and remind yourself about rarely used options.  

Apptainer uses a command/subcommand syntax. Let's use the following to view the available options and subcommands for the main `apptainer` command.

```
$ apptainer --help

Linux container platform optimized for High Performance Computing (HPC) and
Enterprise Performance Computing (EPC)

Usage:
  apptainer [global options...]

Description:
  Apptainer containers provide an application virtualization layer enabling
  mobility of compute via both application and environment portability. With
  Apptainer one is capable of building a root file system that runs on any
  other Linux system where Apptainer is installed.

Options:
      --build-config    use configuration needed for building containers
  -c, --config string   specify a configuration file (for root or
                        unprivileged installation only) (default
                        "/etc/apptainer/apptainer.conf")
  -d, --debug           print debugging information (highest verbosity)
  -h, --help            help for apptainer
      --nocolor         print without color output (default False)
  -q, --quiet           suppress normal output
  -s, --silent          only print errors
  -v, --verbose         print additional information

Available Commands:
  build       Build an Apptainer image
  cache       Manage the local cache
<snip...>
```

There are options that you can pass directly to `apptainer` and then there are subcommands that you can use after the main `apptainer` command. You can get additional information about subcommands by using the `--help` option with them as well. For now, let's look at the `pull` subcommand.

```
$ apptainer pull --help
```

In this case, we see a short `Usage` string to give you a quick hint on how to use the command, a larger `Description` section with more detailed info, an `Options` section, and an `Examples` section.

Some subcommands (like `cache`) are actually "command groups" and have additional subcommands underneath of them. Options can be of two different types. Some options require arguments after them while others behave as "flags" and are simply set to true once they are passed. 

In the next section we'll run our first container via the `shell` subcommand, so it may be useful to look at the help.

```
$ apptainer shell --help
```

As you can see, there are many, many options. But don't be overwhelmed! Apptainer has intelligent defaults that will work in most cases and we will introduce the most important options later.  

## Downloading Containers with pull

You can find pre-built containers in lots of places. Apptainer can convert and run containers in many different formats, including those built by Docker.

In this class, we'll be using containers primarily from [Docker Hub](https://hub.docker.com/).

There are lots of other places to find pre-build containers too. Here are some examples:

- [NGC](https://ngc.nvidia.com/catalog/all?orderBy=modifiedDESC&pageNumber=3&query=&quickFilter=&filters=), developed and maintained by NVIDIA
- [Quay.io](https://quay.io/), developed and maintained by Red Hat
- [BioContainers](https://biocontainers.pro/#/registry), develped and maintained by the Bioconda group
- Cloud providers like Amazon AWS, Microsoft Azure, and Google cloud also have container registries that can work with Apptainer
- [The Singularity Container Library](https://cloud.sylabs.io/library), developed and maintained by Sylabs.

In the following exercises, we'll use the `pull` command to download some sample container images.

First, make a subdirectory within your home directory for the remaining exercises. 

```
$ mkdir ~/apptainer-class

$ cd ~/apptainer-class
```

The popular `lolcow` container is a simple and fun reference container in the Apptainer world.

```
$ apptainer pull docker://godlovedc/lolcow
```

You now have a new file in your current working directory called `lolcow_latest.sif`

```
$ ls
lolcow_latest.sif
```

This is your container. Or more precisely, it is a Singularity Image Format (SIF) file containing an image of a root filesystem. This image is mounted to your host filesystem (in a new "mount namespace") and then entered when you run an Apptainer command.   

Note that you can specify a different name for your container by providing 2 positional arguments after the `pull` command.

Let's pull this container again using the `oras://` (OCI Registry as Storage) protocol to get a native SIF image from Docker Hub

```
$ apptainer pull lolcow.sif oras://index.docker.io/godlovedc/lolcow:sif

$ ls
lolcow_latest.sif  lolcow.sif
```

We'll break that command down a bit.  First, we gave the `pull` command an additional positional argument `lolcow.sif` to specify how we wanted the container to be named. We used `oras://` instead of `docker://` to specify that we wanted to pull a native SIF image with [the ORAS protocol](https://oras.land/). We had to specify `index.docker.io` since `oras://` does not imply an URI.  And finally, we had to add a tag `:sif` to get the correct image.

If this doesn't make a lot of sense right now, don't worry too much about it. We'll be covering this registry stuff in greater detail later. 

For now, let's just delete the container with the longer name and stick with `lolcow.sif`

```
$ rm lolcow_latest.sif
```

## Entering Containers with shell

Now let's enter our new container and look around. We can do so with the `shell` command.

```
$ apptainer shell lolcow.sif 
```

Depending on the environment of your host system you may see your shell prompt change. Let's look at what OS is running inside the container.

```
Apptainer> cat /etc/os-release 
NAME="Ubuntu"
VERSION="18.04.6 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.6 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```

No matter what OS is running on your host, your container is running Ubuntu 18.04 (Bionic Beaver)!


> **_NOTE:_** In general, the Apptainer action commands (like `shell`, `run`, and `exec`) are expected to work with URIs like `docker://` and `oras://` the same as they would work with a local image.

Let's try a few more commands:

```
Apptainer> whoami
godloved

Apptainer> hostname
ciqbox
```

This is one of the core features of Apptainer that makes it so attractive from a security and usability standpoint.  The user remains the same inside and outside of the container. And there is intelligent integration between the container and the host system. 

Regardless of whether or not the program `cowsay` is installed on your host system, you have access to it now because it is installed inside of the container:

```text
Apptainer> cowsay moo
 _____
< moo >
 -----
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

We'll be getting a lot of mileage out of this silly little program as we explore Linux containers.  

This is the command that is executed when the container actually "runs":

```text
Apptainer> fortune | cowsay | lolcat
 ____________________________________
/ A horse! A horse! My kingdom for a \
| horse!                             |
|                                    |
\ -- Wm. Shakespeare, "Richard III"  /
 ------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

More on "running" the container in a minute. For now, don't forget to `exit` the container when you are finished playing!

```text
Apptainer> exit
exit
```

## Executing Containerized Commands with exec

Using the `exec` command, we can run commands within the container from the host system.  

```text
$ apptainer exec lolcow.sif cowsay 'How did you get out of the container?'
 _______________________________________
< How did you get out of the container? >
 ---------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

In this example, apptainer entered the container, ran the `cowsay` command with supplied arguments, displayed the standard output on our host system terminal, and then exited. 

## "Running" a container with (and without) run

As mentioned several times you can "run" a container like so:

```text
$ apptainer run lolcow.sif
 _________________________________________
/ Q: How many Bell Labs Vice Presidents   \
| does it take to change a light bulb? A: |
| That's proprietary information. Answer  |
| available from AT&T on payment of       |
\ license fee (binary only).              /
 -----------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

So what actually happens when you run a container? There is a special file within the container called a `runscript` that is executed when a container is run. You can see this (and other meta-data about the container) using the inspect command.  

```sh
$ apptainer inspect --runscript lolcow.sif
#!/bin/sh

    fortune | cowsay | lolcat
```

In this case the `runscript` consists of three simple commands with the output of each command piped to the subsequent command. 

Because Apptainer containers have pre-defined actions that they must carry out when run, they are actually executable. Note the default permissions when you download or build a container:

```text
$ ls -l lolcow.sif
-rwxr-xr-x 1 student student 93574075 Feb 28 23:02 lolcow_latest.sif
```

This allows you to run execute a container like so:

```text
$ ./lolcow.sif
 ________________________________________
/ It is by the fortune of God that, in   \
| this country, we have three benefits:  |
| freedom of speech, freedom of thought, |
| and the wisdom never to use either.    |
|                                        |
\ -- Mark Twain                          /
 ----------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```
As we shall see later, this nifty trick can makes it easy to forget your applications are containerized and just run them like any old program.  

## Pipes and redirection

Apptainer does not try to isolate your container completely from the host system.  This allows you to do some interesting things. For instance, you can use pipes and redirection to blur the lines between the container and the host system.  

```text
$ apptainer exec lolcow.sif cowsay moo > cowsaid

$ cat cowsaid
 _____
< moo >
 -----
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

We created a file called `cowsaid` in the current working directory with the output of a command that was executed within the container.  >_shock and awe_

We can also pipe things _into_ the container (and that is very tricky).

```text
$ cat cowsaid | apptainer exec lolcow.sif cowsay -n
 ______________________________
/  _____                       \
| < moo >                      |
|  -----                       |
|         \   ^__^             |
|          \  (oo)\_______     |
|             (__)\       )\/\ |
|                 ||----w |    |
\                 ||     ||    /
 ------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

We've created a meta-cow (gossip cow{?} a cow that talks about cows). ;-P

So pipes and redirects work as expected between a container and the host system. If, however, you need to pipe the output of one command in your container to another command in your container, things are slightly more complicated. Pipes and redirects are shell constructs, so if you don't want your host shell to interpret them, you have to hide them from it.

```text
$ apptainer exec lolcow.sif sh -c "fortune | cowsay | lolcat"
```

The above invokes a new shell, but inside the container, and tells it to run the single command line `fortune | cowsay | lolcat`.

## Getting Information About a Container With the inspect Command

We glossed over the `inspect` command above when we used it to see the runscript for our lolcow container.  

```sh
$ apptainer inspect --runscript lolcow.sif
#!/bin/sh

    fortune | cowsay | lolcat
```

This is a script that is saved in the container's metadata that tells it what to do when you run it.  

There are also some metadata that define the environment that the container should run it. You can see this by passing the `--environment` option.

```sh
$ apptainer inspect --environment lolcow.sif 
=== /.singularity.d/env/10-docker2singularity.sh ===
#!/bin/sh
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

=== /.singularity.d/env/90-environment.sh ===
#!/bin/sh
# Copyright (c) Contributors to the Apptainer project, established as
#   Apptainer a Series of LF Projects LLC.
#   For website terms of use, trademark policy, privacy policy and other
#   project policies see https://lfprojects.org/policies
# Copyright (c) 2018-2021, Sylabs Inc. All rights reserved.
# This software is licensed under a 3-clause BSD license. Please consult
# https://github.com/apptainer/apptainer/blob/main/LICENSE.md regarding your
# rights to use or distribute this software.

# Custom environment shell code should follow

    export LC_ALL=C
    export PATH=/usr/games:$PATH
```

This is slightly more complicated because there are several different files that are sourced in sequence to create the environment. 

If we have a native SIF file like we downloaded via the ORAS protocol, we can use the `--deffile` flag to see how the container was built. 

```
$ apptainer inspect --deffile lolcow.sif 
bootstrap: docker
from: ubuntu:18.04

%environment
    export LC_ALL=C
    export PATH=/usr/games:$PATH

%runscript
    fortune | cowsay | lolcat

%post
    apt-get -y update
    apt-get -y install fortune cowsay lolcat
```

We'll return to def files (short for definition files) later when we discuss building containers. 

That covers the basics on how to download and use pre-built containers!  In the coming sections we'll start learning more about container registries and then talk about how to build your own containers.

---
<- [previous](/00-introduction/README.md) - [home](/README.md) - [next](/02-the-ecosystem/README.md) ->
