---
layout: post
title: Building containers from scratch with container tools project
tags: podman buildah skopeo containers-project
---

As container runtime and image standards have evolved and standards developed by projects like the [Open Container Initiative](https://opencontainers.org/) (OCI), there has been an increase in the number of solutions and/or use case specific tools being developed both in FOSS and commercial tools. 

An example of this is the `crun` runtime that we used in the [Centos 8 Stream Kubernetes tutorial](https://dalethestirling.github.io/Lightweight-Kubernetes-Centos-8-Stream/) from earlier this year. This runtime is a memory optimised lightweight OCI compliant runtime and does not have the full lifecycle tooling that other runtimes like Docker. 

This is where the [container tools project](https://github.com/containers) comes in, this provides a number of tools that provide coverage across the build, local run and inspection of OCI compliant container images. In fact, leveraging these tools is becoming the de-facto approach for new projects. 

In this article, we will build a container from scratch using `buildah`, running this container on Linux using `podman` and `skopeo` to inspect the container we have built. 

The examples below were tested on a Centos VirtualMachine running Centos 8 Stream and the tools can be installed using yum.

```
sudo yum install -y podman buildah skopeo
```

First, we are going to stop the docker service to ensure that we are using the tool we have installed.

```
sudo systemctl stop docker
```

The next step is to start the container build, lets's build the container from an empty context using `from scratch` as the base for the container context. 
Then the container context needs to be mounted to the host's filesystem to allow components to be installed and configured. 

You will need to be `root` to mount the container to the host. I escalate my privileges using `sudo -s` so I can assign the container ID and mount path to variables.

In a more secure implementation, you may want to manage your privileges per command.

``` 
newcontainer=$(buildah from scratch)
scratchmnt=$(buildah mount $newcontainer)
```

Now there is an empty container context we can access and start to add some software too. As we want to optimise the size of the container as well as leverage the Centos package manager we are going to:
	
- Manage the dependencies we install by setting `install_weak_deps=false`
- Not install any docs using the option `tsflags=nodocs`
- Limit the language files we install to one using option `override_install_langs=en_US.utf8`

This is done using the `--setopt` parameter in the yum commands as packages are installed. 

This can be taken a step further by separating the installation of `bash` and `coreutils` from the python packages we are going to install. 

By taking these 2 actions in our container build we were able to bring the size of the container down from 642 MB to 449 MB by minimising the number of dependency hops that the `yum` package manager needs to follow.
 
```
yum install --installroot $scratchmnt bash coreutils --releasever 8 --setopt=install_weak_deps=false --setopt=tsflags=nodocs --setopt=override_install_langs=en_US.utf8 -y

yum install --installroot $scratchmnt python3 python3-pip --releasever 8 --setopt=install_weak_deps=false --setopt=tsflags=nodocs --setopt=override_install_langs=en_US.utf8 -y
```

This can be taken a step further by cleaning up the cache files from the `yum` packages that have just been installed. This has shrunk the container from 449 MB to 332 MB.

```
yum clean --installroot $scratchmnt all
```
You can verify this using `du` on the mounted filesystem just like you would any other path mounted to your host. 

```
du -hs $scratchmnt
```

As long as you continue to use the additional `yum` options that were used above. You can keep adding to the python container as you need to in an optimised manner. 

Alternatively, you can also run commands as you would using the `RUN` statement in a Dockerfile.

```
buildah run $newcontainer pip3 install flask
```

Now that we have `flask` installed we need to define the entry point for our container when it is executed.

[Flask](https://flask.palletsprojects.com/en/2.1.x/) is a lightweight web framework that has a huge community following and a massive plugin library making it a great choice for web application use cases.

As we have the container locally we can now develop locally within the container context using `chroot` to set `$scratchmnt` as our new `/` directory.

```
chroot $scratchmnt
```

Now our container context is the root and we can run `flask` using `flask run` this will raise an error as we have no application for the framework to run, so run the command to `exit` to get back to the server context.

Next is to create a `flask` application. This can be done anywhere on the server so let's go to the user home directory on the Linux machine `cd ~`. From here create a directory called `app` and a file called `app.py` with the content using your favourite editor. 

The content for `app.py` is the following. This is the Flask projects Hello World web application. 

```
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return 'Web App with Python Flask!'
```

Now that this is created we can simply copy this to the root of the container filesystem mounted to the host using the `cp` command.

`cp -r ~/app $scratchmnt`

Now you can test the Flask application by using `chroot` again to bind to the container, using the same command as above. 

Now the console is bound to the container context again you can test your Flask web application. 

```
FLASK_APP=./app/app.py flask run
```

This will generate the following output.

```bash-4.4# FLASK_APP=./app/app.py /usr/local/bin/flask run
 * Serving Flask app './app/app.py' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

This is where we start to see what is powerful about `buildah` as not only have all of the dependencies been tested but all of the components for the entry point script have been identified and tested in the container as we are developing it locally.

Now let's put everything together and get the container built so we can run it in `podman`.  

Let's create another file in the user home directory called `entrypoint.sh` and put the following content in it with your favourite editor.

```
#! /bin/bash

export FLASK_APP=/app/app.py
/usr/local/bin/flask run
``` 

Now the entry point script is created it needs to be made executable and then placed within the container context. This could be done using `cp` as we did to move the web app directory, but instead, we can use `buildah` to perform the same action as a `COPY` statement in a Dockerfile. 

```
chmod775 ./entrypoint.sh
buildah copy $newcontainer ~/entrypoint.sh /
```

To validate that the entry point is behaving as expected we can use `buildah	run` again. As this executes a command in real-time rather than at build time like the `RUN` statement in a Dockerfile this can be used to quickly test commands within the container context.

``` 
buildah run $newcontainer /entrypoint.sh
```

This should output the same as was seen above in the earlier execution of the Flask server. Use `CTRL+C` to exit the server running. 

Now that components are behaving as expected the container context needs to be told how to start, this is done by setting parameters in the configuration of the container.

```
buildah config --entrypoint /entrypoint.sh $newcontainer
```

Now the container will start the Flask web application server when an instance of the container image is deployed. Let's set some metadata so that the context of the container can be understood. In the config, let's set values for the author, who it is created by and add the label `name` to identify the image.

```
buildah config --author "Dale Stirling" --created-by "dalethestirling" --label name=python3-flask-demo $newcontainer
```

With the values set in the configuration, these can be verified using the inspect sub command. 

```
buildah inspect $newcontainer
```

In the output not only can you see the definition of the entry point script to be run and the metadata that was set, but also the Capabilities that the container can leverage as well as environment variables within the container.  

Now the container is defined it is time to commit the changes that have been made within the context to be committed to the image. To do this the container image needs to be unmounted from the host and then the changes are added to an image. 

``` 
buildah unmount $newcontainer
buildah commit $newcontainer python3-flask-demo
``` 

Podman is the CLI interface for deploying containers to `runc` (or other OCI runtime) locally. The `podman` command has the same semantics as `docker` to allow for better cross-compatibility.

Using `podman` we can now list images that are cached locally on the host. 

```
podman images
```

You should now see `localhost/python3-flask-demo` in the list of containers.

Having the image built we can inspect the image using either `skopeo` or `podman`.

```
skopeo inspect containers-storage:localhost/python3-flask-demo
podman inspect localhost/python3-flask-demo
```
Through the output of either of these commands, we can see another one of the benefits of using `buildah` in this way. After all of the commands, we have run our container is still a single layer. 

```
...

    "Layers": [
        "sha256:e5bed93f30ac57a4965e8e8ba5facda6208fcc412c5c0034e7d16f07d34c51cf"
    ],
    
...    
```

This can be especially handy if you are extending a base container that contains several layers already. 

Optimising the number of layers in your overlay filesystem can provide performance gains in reading operations as it has to search for the file down through the layers. 

Using `skopeo` also allows you to interrogate details about an image in remote repositories such as Docker hub.

```
skopeo inspect docker://docker.io/dalethestirling/python-demo:latest
```

Now we are at the point where an instance of the container can be created and run. Podman follows in line with the parameter and argument conventions used by `docker` allowing for ease of use.

We will need to expose a port so that we can connect to the `flask` web server and verify it is working. In the same way as you would with `docker` this can be done using the `-e` parameter to expose port 5000.

```
podman run -p 5000:5000 localhost/python3-flask-demo
curl http://127.0.0.1:5000
```
Now you have a working container that you have built from scratch. This container is a single layer and built from the latest OS packages. 

Using the container tools project provides a way to build an OCI compliant container using many existing Linux tools for configuration that can be easily wrapped up into a bash script or similar.  

While this demo is rudimentary. Our container is only 344 MB including `flask` and its dependencies are not far off the RedHat UBI python image at 297.8 MB. This is a 46 MB difference. 

When it comes to automating and testing this approach you can leverage the same tools you would when testing your application locally on your laptop and it can be done inline within your container context as you build the container, but more on that in future posts. 

I have included all of the commands in a [Gist here](https://gist.github.com/dalethestirling/115c297689a0423a96a274f97452769b) to get you started with scripting your container builds with these tools.
