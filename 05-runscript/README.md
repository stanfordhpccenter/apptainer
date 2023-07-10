<- [previous](/04-bind-mounts/README.md) - [home](/README.md) - [next](/06-installation/README.md) ->

---
# Combining Concepts to Create a Container for "Data Analysis"

### Objective

- pull multiple concepts together (building containers, runscripts, bind-mounts) to understand how to create containers that take data as input and produce data as output

## A Common Pattern of Data Analysis

Consider an application that takes a file as input, analyzes the data in the file, and produces another file as output. This is obviously a very common situation.

Let's imagine that we want to use the cowsay program in our `lolcow.sif` to "analyze data".  We should give our container an input file, it should reformat the text (in the form of a cow speaking), and it should dump the output into another file. 

>**_NOTE:_** This is (admittedly) a somewhat ridiculous example. But hopefully, it will illustrate how you can create containers to manipulate some input and produce output.

Here's an example.  First I'll make some "data"

```text
$ echo "These words were spoken by a cow." > /data/input
```

Now I'll "analyze" the "data". (if you have completed the previous examples you should have the `lolcow.sif` file in the directory already.)

```text
$ cd ~/apptainer-class

$ cat /data/input | apptainer exec lolcow.sif cowsay >/data/output
```

The "analyzed data" is saved in a file called `/data/output`. 

```text
$ cat /data/output 
 ___________________________________
< These words were spoken by a cow. >
 -----------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

```

This _works..._ but the piping and redirection syntax is ugly and difficult to remember.  

As discussed before, Apptainer supports a neat trick for making a container function as though it were an executable.  We need to create a **runscript** inside the container. If you remember from previous example our lolcow.def file already contains a runscript.  It causes our container to print a cow with a fortune.  

```text
$ apptainer inspect --runscript lolcow.sif 
#!/bin/sh

    fortune | cowsay | lolcat

$ ./lolcow.sif 
 ___________________________________
/ You have literary talent that you \
\ should take pains to develop.     /
 -----------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

```

Let's rewrite this runscript in the definition file and rebuild our container so that it does something more useful. 

```
bootstrap: docker
from: ubuntu:18.04

%environment
    export LC_ALL=C
    export PATH=/usr/games:$PATH

%post
    apt-get -y update
    apt-get -y install fortune cowsay lolcat

%runscript
    if [ $# -ne 2 ]; then
        echo "Please provide an input and an output file."
        exit 1
    fi
    cat $1 | cowsay > $2
```

Now we must rebuild our container to install the new runscript.  

```text
$ apptainer build --force lolcow.sif lolcow.def
```

Note the `--force` option which ensures our previous container is completely overwritten.

After rebuilding our container, we can call the `lolcow.sif` as though it were an executable, give it input and output file names.  

```text
$ ./lolcow.sif /data/input /data/output2
/.singularity.d/runscript: 7: /.singularity.d/runscript: cannot create /data/output2: Directory nonexistent
cat: /data/input: No such file or directory
```

Whoops!  

We are no longer piping redirecting standard output into and out of the container, so we need to bind mount the `/data` directory into the container.  It will be convenient to simply set the bind path as an environment variable.  

```text
$ export APPTAINER_BINDPATH=/data

$ ./lolcow.sif /data/input /data/output2

$ ./lolcow.sif /data/vader.txt /data/output3

$ cat /data/output2 /data/output3
 ___________________________________
< These words were spoken by a cow. >
 -----------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
 __________________
< I am your father >
 ------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```
Don't forget to unset the environment variable when you are finished with the example!

```text
$ unset APPTAINER_BINDPATH
```

To summarize, we have written a runscript for our container that will do some very basic error checking and expects the location of an input file and an output file allowing it to "analyze" the data. This is obviously a trivial example, but the sky is the limit. If you can code it, you can make your container do it!  

>**_BONUS Question:_** You will often see this or something similar as a containers runscript.

```
%runscript
    python "$@"
```
>What does the `"$@"` do?

---
<- [previous](/04-bind-mounts/README.md) - [home](/README.md) - [next](/06-installation/README.md) ->