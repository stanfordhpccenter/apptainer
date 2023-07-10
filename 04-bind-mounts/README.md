<- [previous](/03-building/README.md) - [home](/README.md) - [next](/05-runscript/README.md) ->

---
# Accessing Host Files with Bind Mounts


### Objectives

- understand the concept of bind-mounts
- learn which directories are bind-mounted into the container by default and how to prevent this behavior if necessary
- be able to bind mount other directories into the container via CLI options/arguments or environment variables

It's possible to read, create, and modify files on the host system from within the container. This is extremely useful. It means that you can containerize code to perform scientific modeling, analysis, etc., but the data can be read from the host system and the output can be saved to the host system at the end of a run.

## Understanding Bind Mounts

Let's be more explicit. Consider this example. First we'll make a new place to play around. 

```text
$ mkdir -p ~/apptainer-class/bind-examples

$ cd ~/apptainer-class/bind-examples
```

If you completed the previous section on building containers, you should have a container called `lolcow.sif` in the parent directory. (Otherwise you can use `apptainer pull ../lolcow.sif oras://index.docker.io/godlovedc/lolcow:sif` to get the container.)

```text
$ apptainer shell ../lolcow.sif

Apptainer> echo wutini > jawa.txt

Apptainer> cat jawa.txt
wutini

Apptainer> exit

$ ls -l
total 4
-rw-r--r--. 1 user user 7 Apr 19 09:31 jawa.txt

$ cat jawa.txt
wutini
```

Here we shelled into a container and created a file with some text in our current working directory.  Even after we exited the container, the file still existed (and had proper ownership). How did this work?

There are several special directories that Apptainer _bind mounts_ into
your container by default.  These include:

- `$HOME`
- `/tmp`
- `/var/tmp`
- `/proc`
- `/sys`
- `/dev`

Some of these directories are important from a systems perspective and help make the container work properly. Others (like `$HOME`) allow you to read and write files on the host system from within the container. Remember that Apptainer's security model makes you the same user inside the container as you are on the host, so your permissions to and ownership are persevered and enforced properly as a side effect. 

Sometimes having host directories bind mounted into the container is convenient, but other times it is unwanted. For instance, during the section on building we wanted to limit automatic bind-mounts to prevent us from accidentally reading or writing data in our `$HOME` directory during the build. We did this by passing the `--containall` flag.  This flag ensures that the minimum bind-mounts just necessary to make the container function properly are used at runtime. 

## Customizing the Directories that are bind-mounted into the Container

You can specify other directories to bind using the `--bind` option or the environmental variable `$APPTAINER_BINDPATH`

Let's say we want to access a directory called `/data` from within our container. For this example, we first need to create this new directory with some data on our host system.  (This bit assumes you have root privileges.)

```text
$ sudo mkdir /data

$ sudo chown $USER:$USER /data

$ echo 'I am your father' > /data/vader.txt
```

Now let's see how bind mounts work.  First, let's list the contents of `/data` within the container without bind mounting `/data` on the host system to it.

```text
$ apptainer exec ../lolcow.sif ls -l /data
/bin/ls: cannot access '/data': No such file or directory
```

Nothing there! Now let's repeat the same command but using the `--bind` option to bind mount `/data` into the container.

```text
$ apptainer exec --bind /data ../lolcow.sif ls -l /data
total 4
-rw-rw-r-- 1 student student 17 Mar  2 00:51 vader.txt
```

Now a `/data` directory is created in the container and it is bind mounted to the `/data` directory on the host system.  

You can bind mount a source directory of one name on the host system to a destination of another name using a `source:destination` syntax, and you can bind mount multiple directories as a comma separated string. For instance:

```text
$ apptainer shell --bind src1:dest1,src2:dest2,src3:dest3 some.sif
```

If no colon is present, Apptainer assumes the source and destination are identical.  To do the same thing with an environment variable, you could do the following:

```text
$ export APPTAINER_BINDPATH=src1:dest1,src2:dest2,src3:dest3
```

Let's set the environment variable to to bind mount our `/data` directory and see how it works. 

```text
$ export APPTAINER_BINDPATH=/data

$ apptainer shell ../lolcow.sif 

Apptainer> cat /data/vader.txt
I am your father

Apptainer> exit
exit
```

Don't forget to unset the environment variable when you are finished with the example!

```text
$ unset APPTAINER_BINDPATH
```

Leaving that environment variable set or setting it automatically by putting it in your `.bashrc` or something can lead to unexpected behavior and could result in data loss if you make a mistake!  

---
<- [previous](/03-building/README.md) - [home](/README.md) - [next](/05-runscript/README.md) ->
