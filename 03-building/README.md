<- [previous](/02-the-ecosystem/README.md) - [home](/README.md) - [next](/04-bind-mounts/README.md) ->

---
# Building Apptainer Containers

### Objectives

- introduction to container development (Apptainer flow)
- understand the main parts of a definition file 
- building containers from other sources and using the build command to change container format
- signing containers, verifying, and encrypted containers

## Developing a New Container

In this section, we will build a brand new container similar to the lolcow container we've been using in the previous examples.

To build an Apptainer container, you must use the `build` command.  The `build` command installs an OS, sets up your container's environment and installs the apps you need.  To use the `build` command, we need a definition file. A [definition file](https://apptainer.org/docs/user/latest/definition_files.html) is a set of instructions telling Apptainer what software to install in the container.

But how do you develop a definition file? Many users find that it is easiest to develop a definition iteratively and interactively. We are going to use a standard development cycle (sometimes referred to as Apptainer flow) to create this container. It consists of the following steps:

- create a writable container (called a `sandbox`)
- shell into the container with the `--writable`, `--fakeroot`, and `--containall` options and tinker with it interactively
- record changes that we like in our definition file
- iteratively rebuild the sandbox from the definition file
- rinse and repeat until we are happy with the result
- rebuild the container from the final definition file as a read-only SIF file for use in production

The Apptainer source code contains several example definition files in the `/examples` subdirectory.  We can look at those to get some ideas about how to build this container. Have a close look at the definition file for the program called asciinema [here](https://github.com/apptainer/apptainer/blob/main/examples/asciinema/Apptainer).  If you are able, you might want to open a browser and keep it there to refer back to as we work. You can also check the [documentation](https://apptainer.org/docs/user/latest/definition_files.html) for an explanation of these sections.

Once we've studied that a bit, let's create a subdirectory and start messing around there. 

```text
$ mkdir -p ~/apptainer-class

$ cd ~/apptainer-class

$ nano lolcow.def # or whatever text editor you like
```

The definition file starts with a header which defines a bootstrap agent, and some keywords appropriate for that agent and then moves on to sections that begin with `%`.  Let's create a header similar to the one in the asciinema example.  We'll give it a tag other than latest. 

```
BootStrap: docker
From: ubuntu:22.04
```

Now let's use this definition file as a starting point to build our `lolcow.img` container. We're going to build using the `--sandbox` option at first so that we can shell in and make changes. A SIF file is immutable, but a sandbox is just a directory.

```text
$ apptainer build --sandbox lolcow lolcow.def
```

This is telling Apptainer to build a container called `lolcow` from the `lolcow.def` definition file. The `--sandbox` option in the command above tells Apptainer that we want to build a special type of container (called a sandbox) for development purposes. 

When your build finishes, you will have a basic Ubuntu container saved in a local directory called `lolcow`.

### Using shell --writable to explore and modify containers

Now let's enter our new container and look around.  

```text
$ apptainer shell lolcow
```

Depending on the environment on your host system you may see your prompt change. 

Let's try installing some software. I used the programs `fortune`, `cowsay`, and `lolcat` to produce the container that we saw in the first demo.  First, let's just update the container with `apt`.

```text
Apptainer> sudo apt-get update
bash: sudo: command not found
```

Whoops!

The `sudo` command is not found. But even if we had installed `sudo` into the container and tried to run this command with it, or change to root using `su`, we would still find it impossible to elevate our privileges within the container.  For example:

```text
Apptainer> sudo apt-get update
sudo: The "no new privileges" flag is set, which prevents sudo from running as root.
sudo: If sudo is running in a container, you may need to adjust the container configuration to disable the flag.
```

This error message is basically telling us "buzz off! you can't elevate privs in the container!". Once again, this is an important concept in Apptainer.  If you enter a container without root privileges, you are unable to obtain root privileges within the container.  This insurance against privilege escalation is the reason that you will find Apptainer installed in so many HPC environments.  

Let's exit the container and re-enter as fakeroot!

```text
Apptainer> exit

$ apptainer shell --fakeroot --writable --containall lolcow
WARNING: Skipping mount /etc/localtime [binds]: /etc/localtime doesn't exist in container
```

That warning is no cause for concern.  

Now we are the fakeroot inside the container which means that it will appear that we are UID 0 and we can do things like use the package manager. The `--fakeroot` option is important and will be explained in more detail later. 

Note also the addition of the `--writable` option.  This option allows us to modify the container.  The changes will actually be saved into the container and will persist across uses. Also note the `--containall` option.  This is a safety consideration to prevent us from accidentally making changes on the host (like in our `$HOME` directory).  We'll talk more about that option when we discuss bind mounts. 

Let's try installing our software again.

```text
Apptainer> apt-get update

Apptainer> apt-get install -y fortune cowsay lolcat
```

Now you should see the programs successfully installed.  Let's try running the demo in this new container.

```text
Apptainer> fortune | cowsay | lolcat
bash: lolcat: command not found
bash: cowsay: command not found
bash: fortune: command not found
```

Drat! 

It looks like the programs were not added to our `$PATH`.  Let's find them, add them, and try again.

```text
Apptainer> find / -type f -executable -name fortune 2>/dev/null
/usr/games/fortune

Apptainer> find / -type f -executable -name cowsay 2>/dev/null
/usr/games/cowsay

Apptainer> find / -type f -executable -name lolcat 2>/dev/null
/usr/games/lolcat
```

These commands are using `find` to look _everywhere_ (starting at `/`) for executable files with the names `fortue`, `cowsay`, and `lolcat`, and ignoring errors (sending them to `/dev/null`). So our new programs were all installed in `/usr/games`

```text
Apptainer> export PATH=$PATH:/usr/games

Apptainer> fortune | cowsay | lolcat
 ________________________________________
/ Keep emotionally active. Cater to your \
\ favorite neurosis.                     /
 ----------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Great!  Things are working properly now.  

> **_NOTE:_** If you receive warnings from the Perl language about the `locale` being incorrect, you can usually fix them with `export LC_ALL=C`.


We changed our path in this session, but those changes will disappear as soon as we exit the container just like they will when you exit any other shell.  To make the changes permanent we should add them to the definition file and re-bootstrap the container.  We'll do that in a minute.

### Building the final production-grade SIF file

Although it is fine to shell into your Apptainer container and make changes while you are debugging, you ultimately want all of these changes to be reflected in your definition file.  Otherwise if you need to reproduce it from scratch you will forget all of the changes you made. You will also want to rebuild you container into something more durable, portable, and robust than a directory.  

Let's update our definition file with the changes we made to this container.

```text
Apptainer> exit

$ nano lolcow.def
```

Here is what our updated definition file should look like.

```
Bootstrap: docker
From: ubuntu:22.04

%post
    apt-get update
    apt-get -y install fortune cowsay lolcat

%environment 
    export PATH=/usr/games:$PATH
    export LC_ALL=C

%runscript
    fortune | cowsay | lolcat
```

Let's rebuild the container with the new definition file.

```text
$ apptainer build lolcow.sif lolcow.def
```

Note that we changed the name of the container.  By omitting the `--sandbox` option, we are building our container in the standard Apptainer file format (SIF).  We are denoting the file format with the (optional) `.sif` extension.  A SIF file is compressed and immutable making it a good choice for a production environment.

Now to make sure it works:

```text
$ ./lolcow.sif 
 _______________________________________
< You will inherit millions of dollars. >
 ---------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Woohoo!! 

As we saw in the previous section when we used the `inspect` command to read the `runscript`, Apptainer stores a lot of [useful metadata](https://apptainer.org/docs/user/latest/environment_and_metadata.html#container-metadata). For instance, if you want to see the definition file that was used to create the container you can use the `inspect` command like so:

```
$ apptainer inspect --deffile lolcow.sif 
Bootstrap: docker
From: ubuntu:22.04

%post
    apt-get update
    apt-get -y install fortune cowsay lolcat

%environment 
    export PATH=/usr/games:$PATH
    export LC_ALL=C

%runscript
    fortune | cowsay | lolcat
```

## Building Containers From other Sources

In the preceding section we used the syntax `Bootstrap: docker` in our definition file header to build our container from a base container on Docker Hub.  But, as long as we have the program `debootstrap` installed, on the host we could also do something like the following to build our base container directly from one of the host OS mirrors. 

```
BootStrap: debootstrap
OSVersion: stable
MirrorURL: http://ftp.us.debian.org/debian/
```

This uses the program [`debootstrap`](https://wiki.debian.org/Debootstrap) to build the root file system using a mirror URL. In this case, we supply a URL that is maintained by Debian. We could also use an Ubuntu URL since it is a derivative of Debian and can also be built with the `debootstrap` program. 

If we wanted to build a CentOS container from the distribution mirror we could use the `yum` package manager similarly. There are actually a ton of different ways to build containers. See this list of ["bootstrap agents"](https://apptainer.org/docs/user/latest/appendix.html#build-modules) in the Apptainer docs.

You can also build a container from a base container on your local file system.  

```
Bootstrap: localimage
From: ~/debian.sif
```

These methods can also be called _without_ providing a definition file using the following shorthand. This syntax is different but it is essentially the same as using the `pull` command. 

```text
$ apptainer build debian1.sif docker://debian

$ apptainer build debian2.sif debian1.sif

$ apptainer build --sandbox debian3 debian2.sif
```

Behind the scenes, Apptainer creates a small definition file for each of these commands and then builds the corresponding container as you can see if you use the `inspect --deffile` command.  

```text
$ apptainer inspect --deffile debian1.sif
bootstrap: docker
from: debian

$ apptainer inspect --deffile debian2.sif
bootstrap: localimage
from: debian1.sif

$ apptainer inspect --deffile debian3
bootstrap: localimage
from: debian2.sif
```

Note that the third command above is actually converting the container type from a SIF file to a sandbox. You can use `build` in this way to convert a SIF file to a sandbox and back again:

```text
$ apptainer build --sandbox deb-sand debian.sif

$ apptainer build deb.sif deb-sand/
```

This can be a useful trick during container development. But it can also produce a container with an uncertain build history if it is misapplied because the changes made to the sandbox will not be reflected in the containers definition file.  It is therefore considered a best practice to build you final production containers from definition files and not to covert them from sandboxes with manual installation steps performed inside.

## Signing your Containers After Building them

You should strongly consider creating a PGP key and using it to sign containers that you build. This way others can check your containers and ensure that they have been built by you.  And you and others can both verify that the containers have not been tampered with or altered in any way when you use them in the future. 

You can generate a new PGP key with the `key newpair` command and push the public key material to the key server (https://keys.openpgp.org by default).  

```text
$ apptainer key newpair
Enter your name (e.g., John Doe) : class instructor
Enter your email address (e.g., john.doe@example.com) : instructor@mymail.com
Enter optional comment (e.g., development keys) : throw away key for demo
Enter a passphrase :
Retype your passphrase :
Generating Entity and OpenPGP Key Pair... done

$ apptainer key list
Public key listing (/home/godloved/.apptainer/keys/pgp-public):

0) U: class instructor (throw away key for demo) <instructor@mymail.com>
   C: 2023-04-18 21:47:56 -0600 MDT
   F: 751B63CEAAFE8B70084E9F1A59B91CCBB4CC9A28
   L: 4096
   --------

$ apptainer key push 751B63CEAAFE8B70084E9F1A59B91CCBB4CC9A28
INFO:    Key server response: Upload successful. This is a new key, a welcome email has been sent.
public key `751B63CEAAFE8B70084E9F1A59B91CCBB4CC9A28' pushed to server successfully
```

This lets you cryptographically sign the container you just created with the `sign` command:

```text
$ apptainer sign lolcow.sif
Signing image: lolcow.sif
Enter key passphrase :
Signature created and applied to lolcow.sif
```

Then you can `push` the SIF file anywhere you like. In the future when you or others `pull` the container you/they can use the `verify` command to make sure that it has not been tampered with.

```text
$ apptainer verify lolcow.sif
Verifying image: lolcow.sif
[LOCAL]   Signing entity: class instructor (throw away key for demo) <instructor@mymail.com>
[LOCAL]   Fingerprint: 751B63CEAAFE8B70084E9F1A59B91CCBB4CC9A28
Objects verified:
ID  |GROUP   |LINK    |TYPE
------------------------------------------------
1   |1       |NONE    |Def.FILE
2   |1       |NONE    |JSON.Generic
3   |1       |NONE    |JSON.Generic
4   |1       |NONE    |FS
Container verified: lolcow.sif
```

In this example, the public key material is available locally, but if this were not the case, apptainer would just retrieve the key material from the key server and use it to verify the image. 

> **_NOTE:_** Anyone can sign a container. So just because a container is signed, does not mean it should be trusted. Users must obtain the fingerprint associated with a given maintainer's key and compare it with that displayed by the `verify` command to ensure that the container is authentic. After that it is up to the user to decide if they trust the maintainer.  

## Encrypted Containers

You can encrypt your containers so that others can not see their contents. You can do so either with a passphrase or with a PEM file.  The passphrase method is not really considered secure and should only be used for testing.  

It is important to realize that the root user can still see your encrypted containers while they are running by entering the mount namespace or gaining access to the memory your container is using while running.  So encrypting your containers will not protect their contents in an untrusted or a compromised environment. 

Encrypting a container is also a privileged operation. So you must be careful when doing so. The best advice is to only encrypt containers that you know well and thoroughly trust and/or use a throw away VM environment.  

To encrypt your container with a PEM key you must first have a key!  Let's create a new sub-directory and then create a key:

```text
$ mkdir -p ~/apptainer-class/encrypted/keys

$ cd !$
cd ~/apptainer-class/encrypted/keys

$ ssh-keygen -t rsa -b 4096 -m pem -N ''
Generating public/private rsa key pair.
Enter file in which to save the key (/home/godloved/.ssh/id_rsa): rsa
[snip...]
```

Great.  Now we'll convert the key to pem format, and rename the private key to help keep things straight. 

```text
$ ssh-keygen -f ./rsa.pub -e -m pem >rsa_pub.pem

$ mv rsa rsa_pri.pem

$ cd ..
```

This workflow is the opposite of the signing and verification workflow.  You encrypt your container with the public key and decrypt it with the private key.  First I'll grab a container that I built myself from OS sources and signed so that I know I can trust it. 

```text
$ apptainer pull oras://docker.io/godlovedc/alpine:latest
INFO:    Using cached SIF image

$ apptainer verify alpine_latest.sif
Verifying image: alpine_latest.sif
[LOCAL]   Signing entity: David Godlove (production key) <davidgodlove@gmail.com>
[LOCAL]   Fingerprint: B7761495F83E6BF7686CA5F0C1A7D02200787921
Objects verified:
ID  |GROUP   |LINK    |TYPE
------------------------------------------------
1   |1       |NONE    |Def.FILE
2   |1       |NONE    |FS
Container verified: alpine_latest.sif
```

Now I'll build a new encrypted container from the container I just downloaded directly using the build trick mentioned earlier. 

```text
$ sudo apptainer build --pem-path=keys/rsa_pub.pem encrypted.sif alpine_latest.sif
[sudo] password for godloved:
WARNING: 'nodev' mount option set on /tmp, it could be a source of failure during build process
INFO:    Starting build...
INFO:    Verifying bootstrap image alpine_latest.sif
INFO:    Creating SIF file...
```

If I try to run the new `encrypted.sif` container without the secret key I get an error.

```text
$ apptainer shell encrypted.sif
FATAL:   Unable to use container encryption. Must supply encryption material through environment variables or flags.
```

But providing the key works as expected:

```text
$ apptainer shell --pem-path keys/rsa
rsa_pri.pem  rsa.pub      rsa_pub.pem

$ apptainer shell --pem-path keys/rsa_pri.pem encrypted.sif
Singularity> cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.11.5
PRETTY_NAME="Alpine Linux v3.11"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://bugs.alpinelinux.org/"
```

Awesome! 

That was a lot of info!! 

Just to quickly recap this section, we talked about building containers from definition files and introduced a method for developing a new container interactively. Then we talked about getting base containers from different sources with various bootstrap agents. And we ended by discussing ways to sign, verify, and encrypt containers to secure them before pushing them to a public repo! whew!!

---
<- [previous](/02-the-ecosystem/README.md) - [home](/README.md) - [next](/04-bind-mounts/README.md) ->