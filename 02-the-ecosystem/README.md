<- [previous](/01-basic-usage/README.md) - [home](/README.md) - [next](/03-building/README.md) ->

---
# Using OCI Registries with Apptainer

### Objectives

- understand the differences between OCI and SIF container formats
- learn how to `pull` both format types from OCI registries
- introduce the concept of trust as it pertains to containers
- learn how to `push` SIF images to OCI registries 

We've spent some time on the basics of Apptainer usage and touched on a few more advanced topics that we will cover in more detail later. Now let's talk more about OCI registries in general and [Docker Hub](https://hub.docker.com/) in specific.  

[Docker Hub](https://hub.docker.com/) hosts over 100,000 pre-built, ready-to-use containers. We've already talked about pulling containers from Docker Hub, but there are more details you should be aware of.

## SIF and OCI Container Formats

SIF and OCI are the two predominate container formats currently in use. Each has advantages and disadvantages and is better suited to different use cases. 

### The Origins of SIF

When Greg Kurtzer first began developing Apptainer he named it Singularity. Part of the reason for the name Singularity was a reference to the fact that the container format was a single image file (and therefore "singular"). At it's inception, Apptainer used [Ext3](https://en.wikipedia.org/wiki/Ext3) images to save the root file system. These had some drawbacks, and after some time the [SquashFS](https://en.wikipedia.org/wiki/SquashFS) file system was adopted instead. SquashFS is a compressed, read-only file system that is notably used for live CDs and is a great fit for containers. 

During the first few years of Singularity/Apptainer development, a desire for new features such as cryptographic signing and encryption led to a proposal for a new file type that would embed one or more SquashFS images as well as other data like signature blocks and tags into one big binary file with a header and pointers to data partitions. This was the genesis of the Singularity Image Format (SIF) file. 

SIF files are easy to manage since they are just files. And they allow you to do things like make your containers executable, and sign, verify, and encrypt your containers. The main drawback is that each SIF file is unique and if you have a lot of containers they take up a lot of disk space. 

### The Origins of OCI

OCI stands for the [Open Container Initiative](https://opencontainers.org/). About the same time that Singularity/Apptainer was starting to be created Docker approached the Linux Foundation to create an organization that would design and publish standards for Linux containers. Of course the first standards were based primarily on the way that Docker runs and packages containers. OCI began as a specification for container formats and container runtimes, and ultimately grew to encompass a specification for container registries as well. So the adjective "OCI" may be applied to a container format, runtime, or registry.  

For our purposes, it is enough to understand that OCI containers are usually a collection of root file systems saved in tarballs (commonly referred to as "layers") along with some metadata. At runtime, the tarballs/layers are extracted, and combined via overlay to create a single coherent file system. An OCI registry is a specialized repository designed to store and serve these layers.

The OCI format requires some additional management from the container platform because the images must be combined into a container at runtime. So you cannot simply copy a container from one place to another or drag it and drop it in a GUI like you can with SIF images. However, one of the great advantages of OCI images is that different containers may share the same layers leading to data deduplication and saving disk space. This can be especially useful for cloud native applications where disk is at a high premium.  

### Interoperability Between OCI and SIF

Apptainer can use containers in the OCI image format. To do this, it pulls in layers from a registry, extracts and overlays them into a single file system, and converts the result to a SIF file. However it is not possible to perform this operation in reverse because the layers are essentially flattened into a single file system and cannot be recovered. 

Thanks to the [ORAS (OCI Registry as Storage) protocol](https://oras.land/), Apptainer can push and pull SIF images to OCI registries including Docker Hub. This allows you to leverage the advantages of SIF while still distributing your containers via well-known public (or private) OCI registries.  

## Pulling Images from OCI Registries

We've already touched on the basics of pulling containers, but there are many details to discuss. 

### Tags and hashes

First, OCI registries have a concept of a tagged image. Tags make it convenient for developers to release several different versions of the same container. For instance, if you wanted to specify that you need Debian version 9 from Docker Hub, you could do so like this:

```text
$ apptainer pull docker://debian:9
```

Or within a definition file:

```text
Bootstrap: docker
From: debian:9
```

There is a _special_ tag called **latest**.  If you omit the `:<tag>` suffix from your `pull` or `build` command or from within your definition file you will get the container tagged with `latest` by default.  This sometimes causes confusion if the `latest` tag doesn't exist for a particular container and an error is encountered. In that case a tag must be supplied.

Tags are not immutable and may change without warning. For instance, the latest tag is automatically assigned to the latest build of a container in Docker Hub. So pulling by tag (or pulling `latest` by default) can result in your pulling 2 different images with the same command. If you are interested in pulling the same container multiple times, you should pull by the hash. Continuing with our Debian 9 example, this will ensure that you get the same one even if the developers change that tag:

```text
$ apptainer pull docker://debian@sha256:f17410575376cc2ad0f6f172699ee825a51588f54f6d72bbfeef6e2fa9a57e2f
```

## Default entities and collections

Let's think about this command:

```text
$ apptainer pull docker://debian
```

When you run that command there are several default values that are provided for you to allow Apptainer to build an entire URI. This is what the full command actually looks like:

```text
$ apptainer pull docker://index.docker.io/debian:latest
```

This container is being pulled using the protocol `docker://`, from the URI `index.docker.io` (which is default for the `docker://` protocol) and getting the official `debian` image using the tag `:latest` (which is also default). 

Notice that we do not have to specify the owner of this image because it is an official image.  If you try this shorthand version of the command with the `lolcow` container, you will find that it fails:

```text
$ apptainer pull docker://lolcow
FATAL:   While making image from oci registry: error fetching image to cache: failed to get checksum for docker://lolcow: reading manifest latest in docker.io/library/lolcow: errors:
denied: requested access to the resource is denied
unauthorized: authentication required
```

Even though that looks like an authentication error it really means that the container simply does not exist. 

There is no official image in Docker Hub called `lolcow`.  For that container to work properly, you must supply the account owner (`godlovedc`):

```text
$ apptainer pull docker://godlovedc/lolcow
```

Similar to the example above there are some intelligent defaults supplied. Consider the following command. When executed, this is the command that apptainer actually acts on:

```text
$ apptainer pull docker://index.docker.io/godlovedc/lolcow:latest
```

In this example the registry (`index.docker.io`) and the tag (`latest`) are implied. These values may need to be manually supplied for some containers on Docker Hub or to download from different registries like Quay.io. 

For instance, to download a CUDA container (running on Ubuntu) from NVIDIA NGC one would need to do something like this:

```text
$ apptainer pull docker://nvcr.io/nvidia/cuda:9.1-devel-ubuntu16.04
```

In this case, the `docker://` protocol specification is a bit confusing since it really just means that we are getting the container in OCI format. 

## Containers and Trust

When you build and/or run a container, you are running someone else's code on your system. Doing so comes with certain inherent security risks. The blog posts [here](https://medium.com/sylabs/cve-2019-5736-and-its-impact-on-singularity-containers-8c6272b4bce6) and [here](https://medium.com/sylabs/a-note-on-cve-2019-14271-running-untrusted-containers-as-root-is-still-a-bad-idea-245d227d4e02) provide some background on the kinds of security concerns containers can cause.

Container security is a large topic and we cannot cover all of the facets in this class. (We'll dive a bit deeper on day 3). But here are a few general guidelines.  

- Never build or run a container as root on a system that you care about. Use `--fakeroot` or run the container in a disposable VM or cloud instance. 
- Review the `runscript` before you run it.
- Use the `--no-home` and `--contain-all` options when running an unfamiliar container.
- Establish your level of trust with a container.

The last point is particularly important and can be accomplished in a few different ways.  

### Docker Hub Official and Certified images

The Docker team works with upstream maintainers (like the Rocky Enterprise Software Foundation or RESF, Canonical, etc.) to create [**Official** images](https://docs.docker.com/docker-hub/official_images/). They've been reviewed by humans, scanned for vulnerabilities, and approved. 

There are a series of steps that upstream maintainers can perform to produce [**Certified** images](https://docs.docker.com/docker-hub/publish/certify-images/).  This includes a standard of best practices and some baseline testing. 

### Signing and verifying Apptainer images

Apptainer gives image maintainers the ability to cryptographically sign images and downstream users can use builtin tools to verify that these images are bit-for-bit reproductions of the originals.  This removes any dependencies on web infrastructure and prevents a specific type of time-of-check to time-of-use (TOCTOU) attack.  

This model also differs from the Docker model of trust because the decision of whether or not to trust a particular image is left to the user and maintainer. Apptainer does not "vouch" for a particular set of images the way that Docker does. It's up to users to obtain fingerprints from maintainers and to judge whether or not they trust a particular maintainer's image.

## Pushing your Containers to an OCI Registry with ORAS

In a previous section, we pulled a native SIF image from Docker Hub using the ORAS protocol like so:

```text
apptainer pull lolcow.sif oras://index.docker.io/godlovedc/lolcow:sif
```

Of course, if you are able to `pull` native SIF files from OCI registries like Docker Hub, that implies that there is a way to `push` these images as well! 

Once again, if we take Docker Hub as an example, you would first need to get a (free) Docker Hub account so that you could create image repositories. Then you would need to create the image repository on Docker Hub. This is easy to do through the web GUI. 

If we assume that your username on DockerHub is `demouser` and you have created a repository called `foo`, _and_ that you have a SIF file that you want to push called `foo.sif` the commands to push your image to Docker Hub will look like this.

First you would need to authenticate.

```text
$ apptainer remote login --username demouser oras://docker.io
INFO:    SylabsCloud defined both globally and individually, using individual
Password / Token:
```

You would then need to supply either your Docker Hub password, or a token that you've generated via docker or podman or something. 

You would then push your SIF to the OCI registry like so:

```text
$ apptainer push foo.sif oras://docker.io/demouser/foo:latest
```

The tag is required when pushing via the ORAS protocol. You are highly encouraged to `sign` the SIF file before pushing it to a public registry so that others can tell this is really an image that you built (and you can tell this is the image you built and has not been tampered with).  We'll talk about the steps for signing your SIF files when we cover container building.

---
<- [previous](/01-basic-usage/README.md) - [home](/README.md) - [next](/03-building/README.md) ->