<- [previous](/06-installation/README.md) - [home](/README.md)

---
# Advanced Apptainer Usage

### Objectives

- investigate other commonly used sections within a definition file and understand when they can be used
- learn about the container cache, it's location, and how to manage it
- understand how `--fakeroot` maps your UID to root and what it means
- become acquainted with the idea of overlays and learn where you can get more info

In this section we will cover some miscellaneous topics and fill in some information gaps that were excluded from the earlier discussions for the sake of simplicity.

## Advanced Definition Files

In the section on building containers we discussed the def file header, several different bootstrap agents, and the sections `%post`, `%environment`, and `%runscript`.  There are other sections that can be included in def files and different ways in which to structure your builds based on def file syntax. 

The [Apptainer documentation on definition files](https://apptainer.org/docs/user/latest/definition_files.html) is an excellent resource for understanding all of the different things you can do when building your containers. I will not reproduce all of that content here, but I will highlight some of the more widely used def file sections that we have not covered so far. 

#### `%files`

The `%files` section allows you to specify files (source and destination) that should be copied from the host into the container at build time.

Example:

```
%files
    /file1
    /file2 /opt
```

#### `%labels`

This section allows you to add metadata to your container.

Example:

```
%labels
    Author alice
    Version v0.0.1
```

#### `%help`

This allows you to provide some documentation in your container that can be viewed with the `apptainer run-help <container>` command.

Example:

```
%help
    This shows how you could document the 
    usage of your container.
```

#### `%test`

The `%test` section is a scriptlet that will run at the conclusion of your build or when invoked with the `apptainer test` command. It is useful for checking that the container built properly and functions as expected. 

Example:

```
%test
    grep -q NAME=\"Ubuntu\" /etc/os-release
    if [ $? -eq 0 ]; then
        echo "Container base is Ubuntu as expected."
    else
        echo "Container base is not Ubuntu."
        exit 1
    fi
```

### SCIF (Scientific Filesystem) Container Builds 

SCIF is a standard (developed by Apptainer community members) for encapsulating multiple apps within a single container. There are special section keywords (like `%appinstall`, `%apprun`, etc.) that set up a special shell scripts within the container metadata. These scripts are then invoked to trigger specific applications to run within the container when action commands like `run` and `exec` are invoked with the `--app <app name>` option argument pair.   

You can read more about these types of builds [here](https://apptainer.org/docs/user/latest/definition_files.html#scif-apps) and more about the spec [here](https://sci-f.github.io/).

### Multi-Stage builds

Multi-stage builds allow you to build a container in one environment and then copy software from that container to a new container. This allows you to do things like create one container with a compiler and all of the necessary tooling to build you binaries, and then copy the compiled binaries to a new container that provides a lightweight runtime.  

You trigger multistage builds by including more than one header in the same def file with the keyword `Stage:`. The `%files` section is overloaded to handle copying files between build stages.  You can learn more [here](https://apptainer.org/docs/user/1.0/definition_files.html#multi-stage-builds).

### Automatically Generated Definition Files

There are a few HPC-oriented package managers that ease the installation of software by users or administrators. Several of these (e.g. [Spack](https://spack.io/), [Easybuild](https://easybuild.io/), [Guix](https://guix.gnu.org/)) will generate Apptainer definition files for you upon request to allow you to easily build containers based on package specs. 

These container images may be highly optimized for HPC and commonly employ tricks like multi-stage builds mentioned above to make the final containers smaller and more easy to manage. 

## The Apptainer Cache

You may have noticed that Apptainer sometimes downloads containers from sources like Docker Hub, but it sometimes says it uses a cached image instead. By default Apptainer creates a hidden sub-directory called `.apptainer` in the user's home directory the first time it runs and uses it to store things like keys, metadata, etc. It also creates a `cache` subdirectory there and stores artifacts like SIF files and OCI layers (tarballs) so that they can be obtained more quickly without downloading a second time if they are requested again.  

This is particularly important in HPC environments where you may be running the same container across hundreds or thousands of nodes. In these types of environments, you should cache the images to a shared storage location so that multiple nodes don't try to download the same container.  

If your `$HOME` directory does not have a lot of storage space, it may be necessary to move the cache do a different location.  This can be done easily with the `$APPTAINER_CACHEDIR` environment variable.  

It may also become necessary to clean your cache out from time to time to free up disk space. The `cache` command group is useful for this purpose. 

```text
$ apptainer cache list
There are 27 container file(s) using 4.47 GiB and 91 oci blob file(s) using 2.68 GiB of space
Total space used: 7.15 GiB

$ apptainer cache clean
This will delete everything in your cache (containers from all sources and OCI blobs). 
Hint: You can see exactly what would be deleted by canceling and using the --dry-run option.
Do you want to continue? [N/y] y
INFO:    Removing blob cache entry: blobs
INFO:    Removing blob cache entry: index.json
INFO:    Removing blob cache entry: oci-layout
[snip...]

$ apptainer cache list
There are 0 container file(s) using 0.00 KiB and 0 oci blob file(s) using 0.00 KiB of space
Total space used: 0.00 KiB
```

## Using --fakeroot

In the section on building containers we did not build with any elevated privileges (except in the example on encrypted containers) but we were able to leverage root only programs like `dnf` and `apt`.  How?  

Modern Linux kernels support a feature called the user namespace that allows you to map one UID outside of the namespace to another UID inside of the namespace. Recent versions of Apptainer take advantage of that technology by pivoting into a new user namespace when you build a container and mapping your UID on the host system to UID 0 in the container. 

This feature is used implicitly when you build a container but you can take advantage of it explicitly when you run a container too. Consider the following example:

```text
$ pwd
/home/user

$ touch foo

$ ls -l foo
-rw-r--r--. 1 user user 0 Apr 19 13:39 foo

$ apptainer shell --fakeroot docker://alpine
INFO:    Using cached SIF image

Apptainer> pwd
/root

Apptainer> ls -l foo
-rw-r--r--    1 root     root             0 Apr 19 13:39 foo

Apptainer> touch made-by-root

Apptainer> ls -l made-by-root 
-rw-r--r--    1 root     root             0 Apr 19 13:39 made-by-root

Apptainer> exit

$ ls -l made-by-root 
-rw-r--r--. 1 user user 0 Apr 19 13:39 made-by-root
```

Note the illusion above. The user's `$HOME` directory is bind-mounted into the container at `/root`.  Files previous owned by the user appear to be owned by root, and files that appear to be created by root within the container are owned by the user on the host system.

By combining the `--fakeroot` option with the `--writable` or `--writable-tmpfs` options, you can write to places within the container that are root owned, or use tools like package managers that would otherwise be disallowed.  

## Overlays

Although SIF files contain immutable, read-only, compressed squashfs images, you can obtain the illusion of write access by using overlays. Overlays are writable images (ext3) or directories that you invoke at container runtime with the `--overlay` command to layer on top of your container and provide a place to write new data. You an even add a writable overlay to your SIF file itself so that it is stored along with the rest of the container and gives you SIF the appearance of write access.  

You can learn more about creating and using overlays [here](https://apptainer.org/docs/user/latest/persistent_overlays.html). Here is an quick example of adding a persistent overlay to a SIF image and using it to create a few files in the root directory that persist across runs. 

```
$ apptainer pull rocky.sif docker://rockylinux:9

$ apptainer overlay create --size 1024 rocky.sif 

$ apptainer shell --fakeroot --writable rocky.sif 

Apptainer> touch /foo /bar

Apptainer> exit

$ apptainer shell rocky.sif 

Apptainer> ls /foo /bar
/bar  /foo
```

---
<- [previous](/06-installation/README.md) - [home](/README.md)