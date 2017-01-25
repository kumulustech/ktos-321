# Docker use Basics

We have two choices for using Docker int he lab enviornment(s):

1) Use our control or node-1 machine, which already have docker installed
2) Use the "native" Docker service from Docker.com:
```
https://www.docker.com/products/docker
```

Caution: You will want to run the rest of this lab from within this Vagrant VM. If you have the lastest version of Docker installed on you laptop, the lab will still work, but note that the specified paths and IP addresses will differ in some cases.

In either case, we should first validate that we can even talk to the docker daemon:
```
docker ps
```

The output should look something like this:
root@control:/root# docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES

1) Run a docker image
We'll first deploy a simple lightweight container based on the Alpine Linux distribution. This is a great base container as it is very light weight (often < 20MB), and can be easily extended to address most application requirements.

We'll also instruct docker to launch a simple command line shell "in" the running container.
docker run --rm -i -t alpine:latest sh

The command parameters to the docker run command in this case include:

--rm remove resources when the container exits (doesn't delete the downloaded image)
-i interactive move, don't go into the background if that would be normal
-t create a tty/terminal for the executable
alpine:latest is the name of the image and the version tag (latest)
sh an executable to run that already exists in the image.

A few things to try:
- Look at the process table. how does this compare to the process table of the underlying linux instance?
- Look around the file system, is this the same or different from the file system in the base OS?
- Exit the container, and re-run the command with different command interpreters (e.g. try bash). Does it work? What doest the system tell you.

You should start to get a feel for the container environment vs. a normal Linux system.

2) Use an image to launch a simple web server
```
docker run --rm -p 8000:8000 -i -t alpine:latest sh
```

New parameter:
-p expose port 8000 to the system

We can, in the interpreter, add a set of additional resources, like a python interpreter. And we can create files, and run applications.
```
apk update && apk add python3
echo "Hello World!" > index.html
python3 -m http.server 8000
```
open a browser on your local machine and point it to http://192.168.56.10:8000 (or http://localhost:8000 on your laptop)

3) Let's get our web pages from the local system disk rather than creating it
First, create a file to serve in the local directory with a new comment, such as:
```
echo 'Hello Brave New WORLD!' > index.html
```
Then launch our Alpine container again:
```
docker run --rm -p 8000:8000 -v ./:/root -i -t alpine:latest sh
```
```
apk update && apk add python3
cd /root
python3 -m http.server 8000
```
Again, open a browser on your local machine and point it to http://192.168.56.10:8000 (or http://localhost:8000 on your laptop)

4) Let's make this all permanent
Running a container and then modifying it is not the best way to use containers, although we can get an idea of what we have to do to get our code installed properly in the container environment. In fact, it would be better if we could just incorporate our application/service into the image instead.

We will create a Dockerfile with the above commands, and with that description, create an image of our own.

On the docker VM, create a file named Dockerfile in the local directory. You can use a text editor on the VM itself (vi is there by default, or joe editor, which has embedded instructions).

```
FROM alpine:latest
MAINTAINER your-email@example.com
LABEL description='A simple python launched Hello World web server'
RUN apk update && apk add python3
RUN echo "Hello World!" > /root/index.html
WORKDIR /root
EXPOSE 8000
CMD python3 -m http.server 8000
```

We've added a few things beyond what we've talked about in the lecture, namely:
LABEL a key=value parameter to add additional information about your container
WORKDIR the location where commands are expected to run, this will make it easier for us to re-use this container by over-riding the /root directory
MAINTAINER we will want to store this image in the public Docker registry (hub.docker.com), so we want to make sure folks know who built this particular image

Now we can build our container:

```
docker build . -t hello-world:latest
```

. path to the Dockerfile/build directory, in our case, we should be in that directory already, so that's simple enough
-t add a tag, in this case we're tagging it with a name and a version separated by a :

NOTE: we could extend our tag by adding our docker_hub_user name to the name of the image. This way, we could more easily push this image up to the docker hub registry. This is useful if we want to easily re-use this image on other systems. In that case, our -t parameter would look like:

username/image:version

For now, we'll leave the name off, and we can once again run our container, but this time, using the image we just built. Notice that we don't pass an additional command to launch an interpreter, and we've passed the -d parameter to "daemonize" our running instance.
```
docker run -d --name hw -p 8000:8000 hello-world:latest
```
Without having to get into the container we should be able point our web browser at http://192.168.56.10:8000/ (or locahost) and get our default Hello World message:
```
Hello World!
```
We can also run 'curl' from the local machine, which should give us the same response on the command line:
```
curl http://localhost:8000/
```
If we found that things didn't build quite properly, we can still look at the state of our currently running system, using the 'exec' command. Since we passed a name parameter to our run command, we can use it to launch the exec command. Otherwise, we'd have to discover our container ID (or grab it from the output of the run command) in order to tell exec exactly which container we want to exec into.
```
docker exec -it hw sh
```
The image we have now has our python interpreter installed, and has our basic index file in /root. Update the index.html in /root with a different message, for example
```
echo "This is fun!" > /root/index.html
```
Exit the sh interpreter. Does curl return the updated information now?

Stop and remove our running container to clean up:
```
docker ps
```
Gets us a list of running containers
```
docker stop {CONTAINER ID}
```
or
```
docker stop {NAMES}
```

Note: Variables you need to replace with your data will be represented like so : {NAMES}. Replace everything, including the curly braces with your new information.
and then
```
docker rm {CONTAINER ID}
```
or
```
docker rm {NAMES}
```

5) Now we have an image that's functional, includes our code, and is re-usable.

Let's now change the message our web server provides by re-mapping the storage directory (very common for web services):

First stop and remove the previous instance if you haven't already (see the previous step)

Now, let's launch the container and map a local directory to our container's /root directory (remember that we told our image to use this as the working directory):
```
docker run -d -p 8000:8000 -v ./:/root hello-world:latest
```
What do we get when we now run:
```
curl http://localhost:8000
```

7) Docker compose.

If we want to further simply the process of starting a container (or a group of associated containers), we can create a compose description that includes the port mapping and volume mappings we would otherwise have to pass on the command line.

In the local directory, create a file called docker-compose.yml. Note that as a YAML formatted file, spaces are important, do not use <TAB> to space this file unless you are using a text editor that is configured to convert tabs into spaces.
```
version: '2'
services:
hello-world:
build:
context: .
image: hello-world:latest
ports:
- "8000:8000"
volumes:
- ./:/root
```

We can then launch the container (which will also build a new version):
```
docker-compose up -d
```

Much like with the docker command, there are stop and rm commands as well, as well as a ps command.

Discover, stop, and remove the composed resources.


Caution: You are now done using this your environment. Don't forget to run
docker rm or docker-compose rm to clean up your environment. It may otherwise cause conflicts when setting up the Vagrant environment for your Kubernetes lab.
