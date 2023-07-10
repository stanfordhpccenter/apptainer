<- [previous](/05-runscript/README.md) - [home](/README.md) - [next](/07-advanced-usage/README.md) ->

---
# Installing Apptainer as an Unprivileged User

### Objectives

- obtain a basic understanding of the various types of Apptainer installations
- learn how to install an unprivileged, relocatable version of Apptainer in your own space
- verify the installation by building and running a test container 

## Different Types of Apptainer Installations

During normal use, Apptainer carries out operations on a user's behalf that require elevated privileges. This includes things like mounting the container image to the host file system, creating and pivoting into a new mount namespace, bind-mounting host system files and directories into the container, etc. 

### Root-owned set-UID (suid) installation

Historically, Apptainer did this without granting the user any extra permissions by temporarily elevating the processes permissions to carry out these task and then dropping the permissions again.  It did this through a root-owned set-UID program that was installed with Apptainer.  A set-UID or (or suid) program allows a user to execute processes as another user that owns the executable file.  When the executable is owned by root this allows any user to perform actions as root. 

Because part of the Apptainer installation had to be owned by root, the program itself had to be installed by the root user. 

### Unprivileged Installation

There have been advances in the Linux kernel and in related tooling that now allow many operations that were once privileged to be carried out safely by unprivileged users.  Apptainer has leveraged these advances, with the end result that most Apptainer functionality no longer requires a privileged installation on a relatively recent operating system.  

However, since it is typically installed using tools like `yum/dnf` or `apt`, or it is installed in locations owned by root, even unprivileged installations of Apptainer are typically installed by root. 

### Relocatable Unprivileged Installations

Long-time community member and the release manager of Apptainer Dr. Dave Dykstra of Fermi National Accelerator Laboratory (Fermilab) has developed a convenience script for installing Apptainer.  In a nutshell, it allows you to install Apptainer using a very similar process to a package manager, but in a location of your choosing and without any privilege.  The practical result of this work is that you can install your own version of Apptainer on any system where you have write access regardless of your privilege level! 

## Installing Apptainer in Your Own Space

The script to install an unprivileged and relocatable version of Apptainer can be found [here](https://github.com/apptainer/apptainer/blob/main/tools/install-unprivileged.sh). You can download it like so:

```text
$ wget https://raw.githubusercontent.com/apptainer/apptainer/main/tools/install-unprivileged.sh 
```

After downloading it, make it executable and run it without arguments to see the available options.

```text
$ chmod 750 install-unprivileged.sh

$ ./install-unprivileged.sh
Usage: install-unprivileged.sh [-d dist] [-a arch] [-v version] [-o ] installpath
Installs apptainer version and its dependencies from EPEL+baseOS
   or Fedora.
 dist can start with el or fc and ends with the major version number,
   e.g. el9 or fc37, default based on the current system.
   As a convenience, active debian and ubuntu versions (not counting 18.04)
   get translated into a compatible el version for downloads.
   OpenSUSE based distributions are also mapped to el, or native openSUSE
   binaries can be used via the -o switch.
 arch can be any architecture supported by EPEL or Fedora,
   default based on the current system.
 version selects a specific apptainer version, default latest release,
   although if it ends in '.rpm' then apptainer will come from there.
```

Let's create a directory in our `$HOME` and install it there:

```text
$ mkdir -pv ~/apptainer/v1.1.7
mkdir: created directory '/home/user/apptainer'
mkdir: created directory '/home/user/apptainer/v1.1.7'

$ ./install-unprivileged.sh -v 1.1.7 ~/apptainer/v1.1.7/
Extracting https://kojipkgs.fedoraproject.org/packages/apptainer/1.1.7/1.fc36/x86_64/apptainer-1.1.7-1.fc36.x86_64.rpm
208038 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/Packages/l/lzo-2.10-6.fc36.x86_64.rpm
322 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/Packages/l/libseccomp-2.5.3-2.fc36.x86_64.rpm
347 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/updates/36/Everything/x86_64/Packages/s/squashfs-tools-4.5.1-1.fc36.x86_64.rpm
1168 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/Packages/f/fuse-libs-2.9.9-14.fc36.x86_64.rpm
616 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/updates/36/Everything/x86_64/Packages/l/libzstd-1.5.4-1.fc36.x86_64.rpm
1494 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/Packages/e/e2fsprogs-libs-1.46.5-2.fc36.x86_64.rpm
1076 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/Packages/e/e2fsprogs-1.46.5-2.fc36.x86_64.rpm
8747 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/updates/36/Everything/x86_64/Packages/f/fuse-overlayfs-1.9-6.fc36.x86_64.rpm
293 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/releases/36/Everything/x86_64/os/Packages/f/fuse3-libs-3.10.5-2.fc36.x86_64.rpm
567 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/updates/36/Everything/x86_64/Packages/f/fakeroot-1.31-1.fc36.x86_64.rpm
316 blocks
Extracting https://dl.fedoraproject.org/pub/fedora/linux/updates/36/Everything/x86_64/Packages/f/fakeroot-libs-1.31-1.fc36.x86_64.rpm
270 blocks
Patching fakeroot-sysv to make it relocatable
Creating bin/apptainer and bin/singularity
Installation complete in /home/user/apptainer/v1.1.7/
```

Finally, add the appropriate directories to your `$PATH`. The `x86_64/bin` directory provides the tools that make your containers executable. 

```text
$ export PATH=~/apptainer/v1.1.7/bin:~/apptainer/v1.1.7/x86_64/bin:$PATH

$ which apptainer
~/apptainer/v1.1.7/bin/apptainer
```

The bash completion file can be found at `~/apptainer/v1.1.7/x86_64/share/bash-completion/completions/apptainer`. If you source this file (or cause it be be sourced in your `.bashrc` or similar) you can press the tab button to complete apptainer options and commands.  

## Verify the Installation 

First create a def file that has privileged operations in it like this:

```
Bootstrap: docker
From: rockylinux:9

%post
    dnf -y update
    dnf -y install vim git wget
```

Then use it to build a container.

```text
$ apptainer build test.sif test.def
WARNING: 'nodev' mount option set on /tmp, it could be a source of failure during build process
INFO:    Starting build...
Getting image source signatures
Copying blob 9d28f3f24f51 skipped: already exists
Copying config 7b6a06235a done
Writing manifest to image destination
Storing signatures
2023/04/17 22:51:08  info unpack layer: sha256:9d28f3f24f518c7941aff60018249a76337954b52155d8cbb23271acfb6734ae
2023/04/17 22:51:09  warn xattr{usr/bin/write} ignoring ENOTSUP on setxattr "user.rootlesscontainers"
2023/04/17 22:51:09  warn xattr{/tmp/build-temp-2178421358/rootfs/usr/bin/write} destination filesystem does not support xattrs, further warnings will be suppressed
INFO:    Running post scriptlet
+ dnf -y update
Rocky Linux 9 - BaseOS                                                         378 kB/s | 1.8 MB     00:04
Rocky Linux 9 - AppStream                                                      639 kB/s | 6.8 MB     00:10
Rocky Linux 9 - Extras                                                         4.6 kB/s | 8.7 kB     00:01
Dependencies resolved.
===============================================================================================================
 Package                             Architecture      Version                         Repository         Size
===============================================================================================================
Upgrading:
 curl-minimal                        x86_64            7.76.1-19.el9_1.2               baseos            128 k
 gnutls                              x86_64            3.7.6-18.el9_1                  baseos            1.0 M
 libcurl-minimal                     x86_64            7.76.1-19.el9_1.2               baseos            226 k
 libgcrypt                           x86_64            1.10.0-10.el9_1                 baseos            504 k
 lua-libs                            x86_64            5.4.4-2.el9_1                   baseos            214 k
 openssl                             x86_64            1:3.0.1-47.el9_1                baseos            1.1 M
 openssl-libs                        x86_64            1:3.0.1-47.el9_1                baseos            2.1 M
 python3                             x86_64            3.9.14-1.el9_1.2                baseos             27 k
 python3-libs                        x86_64            3.9.14-1.el9_1.2                baseos            7.3 M
 python3-setuptools-wheel            noarch            53.0.0-10.el9_1.1               baseos            468 k
 systemd-libs                        x86_64            250-12.el9_1.3                  baseos            629 k
 tar                                 x86_64            2:1.34-6.el9_1                  baseos            876 k
 tzdata                              noarch            2023c-1.el9                     baseos            433 k
 vim-minimal                         x86_64            2:8.2.2637-20.el9_1             baseos            674 k

Transaction Summary
===============================================================================================================
Upgrade  14 Packages

Total download size: 16 M
Downloading Packages:
(1/14): lua-libs-5.4.4-2.el9_1.x86_64.rpm                                      152 kB/s | 214 kB     00:01
[snip...]
Complete!
INFO:    Creating SIF file...
INFO:    Build complete: test.sif
```

The container has built without issue.  Let's see how it works!

```text
$ apptainer shell --fakeroot test.sif

Apptainer> whoami
root

Apptainer> id
uid=0(root) gid=0(root) groups=0(root),65534(nobody)

Apptainer> which vim
/usr/bin/vim

Apptainer> which wget
/usr/bin/wget

Apptainer> which git
/usr/bin/git
```

Everything seems to work as expected.  Even `--fakeroot`!

---
<- [previous](/05-runscript/README.md) - [home](/README.md) - [next](/07-advanced-usage/README.md) ->
