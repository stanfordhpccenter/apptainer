[home](/README.md) - [next](/01-basic-usage/README.md) ->

---
# Introduction

### Objectives

- understand conceptually the idea and use of Linux containers
- learn the difference between containers and virtual machines and when to favor each
- see how Apptainer compares to other container platforms and where it fits in

## What IS a software container anyway? (And what's it good for?)

A container allows you to stick an application and all of its dependencies into a single package.  This makes your application portable, shareable, and reproducible.

Containers foster portability and reproducibility because they package **ALL** of an applications dependencies... including its own tiny operating system!

This means your application won't break when you port it to a new environment. Your app brings its environment with it.

Here are some ideas of things you can do with containers:

- Package an analysis pipeline so that it runs on your laptop, in the cloud, and in a high performance computing (HPC) environment to produce the same result.
- Publish a paper and include a link to a container with all of the data and software that you used so that others can easily reproduce your results.
- Install and run an application that requires a complicated stack of dependencies with a few keystrokes.
- Create a pipeline or complex workflow where each individual program is meant to run on a different operating system.

> **_NOTE:_** This introduction is intentionally a little over simplified. We'll be covering the topics of reproducibility and portability in great detail in later sections. 

## Containers Vs. Virtual machines (VMs)

Containers and VMs are both types of virtualization.  If you are new to the world of containers (or maybe even if you are not) it is useful to conceptualize containers in relation to VMs, to understand the differences between the two, and know when to use each.

**Virtual Machines** install every last bit of an operating system (OS) right down to the core software that allows the OS to control the hardware (called the _kernel_).  They may also virtualize hardware, meaning that they "create" hardware devices out of software. This means that VMs:

- Are complete in the sense that you can use a VM to interact with your computer via a different OS.
- Are extremely flexible.  For instance you an install a Windows VM on a Mac using software like [VirtualBox](https://www.virtualbox.org/wiki/VirtualBox).  
- Are slow and resource hungry.  Every time you start a VM, keep in mind that it is running its own operating system.

**Containers** share the system hardware and a kernel with the host OS.  This means that Containers:

- Are less flexible than VMs.  For example, a Linux container must be run on a Linux host OS.  (Although you can mix and match distributions.)  In practice, containers are only developed on Linux because they are based on features of the Linux kernel and Linux file systems.
- Are much faster and lighter weight than VMs.  A container may be just a few MB.
- Start and stop quickly and are suitable for running single apps.

Because of their differences, VMs and containers serve different purposes and should be favored under different circumstances.  

- VMs are good for long running interactive sessions where you may want to use several different applications.  (Checking email on Outlook and using Microsoft Word and Excel on your Mac).
- Containers are better suited to running one or two applications, often non-interactively, in their own custom environments.

## Docker and Company

Since many people have used Docker (or one of its successors) it may be informative to compare these types of containers with Apptainer.  

[Docker](https://www.docker.com/) did not invent Linux containers, but during the around 2013-2016 time frame Docker made containers very popular. It was, for a time, the most widely used container software. It has been very influential in shaping the feature set and user interface of other software (such as podman), and it formed the early basis for the Open Container Initiative (a group seeking to standardize and govern container technology across industry). 

Docker and its successors (such as podman) have several strengths and weaknesses making them a good choice for some projects but not for others.

**philosophy**

Docker is built for running multiple containers on a single system and it allows containers to share common software features for efficiency.  It also seeks to fully isolate each container from all other containers and from the host system unless directed to do otherwise.  

Docker running in normal operation assumes that you will be a root user.  Or that it will be OK for you to elevate your privileges if you are not a root user.
See [this documentation](https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface) for some details.

**strengths**

- Has become a de-facto standard for cloud native applications 
- [Docker Hub](https://hub.docker.com/)!
    - A place to build and host your containers
    - Fully integrated into core Docker
    - Hundreds of thousands of pre-built containers
    - Provides an ecosystem for container orchestration

**weaknesses**

- Difficult to learn (comparatively)
    - Details abstracted away from the user (can be seen as a strength) 
    - Complex container model (layers)
- Not architected with security in mind
- Not built for HPC (but good for cloud) 

Docker and company shine for DevOPs teams providing cloud-native applications built on micro-services architecture to users, and for service oriented orchestration (k8s).

## Apptainer

[Apptainer](http://apptainer.org) (originally Singularity) was invented in 2015 by Greg Kurtzer while at Lawrence Berkley National labs and is now developed by the open source community. In 2021 the Singularity project was adopted by the Linux Foundation and changed its name to Apptainer to better reflect its purpose and open-source identity. Apptainer was developed with security as a first principle, and with scientific software and HPC systems in mind.  

**philosophy**

Apptainer users kernel and file system features to enforce the same privileges inside the container as those that the user possesses outside the container. If you enter a container without elevated privileges it is impossible to gain elevated privileges inside. Moreover, you are _you_ inside the container. You have the same user/group names and UID/GID inside and outside the container.

Apptainer assumes ([more or less](http://containers-ftw.org/SCI-F/)) that each application will have its own container.  It does _not_ seek to fully isolate containers from one another or the host system but strives for intelligent integration. 

**strengths**

- Easy to learn and use (relatively speaking)
- Approved for HPC and installed almost everywhere HPC is being done
- Can convert Docker containers to Apptainer and run containers directly from Docker Hub
- Uses Singularity Image Format (SIF) containers which offer a number of advantages such as supporting cryptographic signatures and encryption
- Integration with GPUs, MPI, and batch scheduling systems

**weaknesses**

- A more specialized use case and user community (HPC-oriented)
- Limited OCI compatibility
- Does not benefit from the data deduplication of layers (but SIF has advantages as well)

Apptainer shines for scientific software running in an HPC environment.  

The remainder of this material is meant to be completed as hands-on exercises in a Linux environment. It was developed on Rocky 8, but can similarly be run on other Linux distributions (making appropriate changes to any package manager commands etc.) If you already have access to an installation of Apptainer, you man continue to the next chapter. Otherwise you may want to skip ahead to [the section on installation](/06-installation/README.md) and then return to the next chapter after you have apptainer running.  

---
[home](/README.md) - [next](/01-basic-usage/README.md) ->