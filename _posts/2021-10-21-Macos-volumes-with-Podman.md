---
layout: post
title: Mounting Macos directories in containers with Podman
tags: podman sshfs osx
---

Recently RedHat released an article about [Podman's machine function](https://www.redhat.com/sysadmin/podman-mac-machine-architecture) and how it can be leveraged on Macos. 

So while I was setting up my new M1 MacBook air I thought this would be a good alternative to try of Docker Desktop. 

The experience so far has been good the fact that the running of docker containers is happening in a Qemu Fedora VM does not impact the experience, with one exception Volume mounts. 

While you can use scp to copy up files to the VM this quickly become cumbersome when doing development. 

The alternative to this is to mount the Macos filesystem inside the container. For this I landed on using `sshfs` via an ssh reverse proxy. 

This does take some setting up and will need to be repeated each time you update the machine VM for `podman`. 

First we will focus on getting the VM ready. To do this we need to run a number of commands on the VM host. For some of these to work we will need to set up the reverse proxy we intend to use for the `sshfs` mount, this means that we can't run the `podman machine ssh` command to connect. Instead we need to get the unprivileged port ssh is listening on for the VM. This is done using the following command: 

`podman machine --log-level=debug ssh -- exit 2>&1 | grep Executing | awk {'print $8'}`

Now we will use this port to connect to the host and run the following commends:

```bash
# Connect to the host 
ssh -i ~/.ssh/podman-machine-default -R 10000:$(hostname):22 -p <PORT> core@localhost

# Generate SSH key and share back to Macos host
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id -p 10000 <MAC USERNAME>@127.0.0.1

# Create mount point for host filesystem
sudo mkdir -p /mnt/Users
sudo chown core:core /mnt/Users
```

Now we have the ability to run the `sshfs` command and mount the drive across the reverse proxy link. 

`sshfs -p 10000 $USER@127.0.0.1:/Users /mnt/Users`

In this scenario we would need to keep the terminal window open and un interrupted for the mount to remain in place. Though we can use tools to put this in the background. 

To do the we need to `CTRL-C` out of the `sshfs` command and `exit` the shell to the VM. 

No we are in the mac terminal we can use `screen` to create a new tty that we can put this interactive shell in the background and leave running through the following commands:

```bash
# Define our screen session
screen -t podman

# Connect to the VM with reverse proxy
ssh -i ~/.ssh/podman-machine-default -R 10000:$(hostname):22 -p <PORT> core@localhost

# Mount the host filesystem to the guest
sshfs -p 10000 $USER@127.0.0.1:/Users /mnt/Users 
```

Now that `sshfs` is running you can move the tty to the background by detaching from it with `CTRL-A + D`.

With this complete you can now mount volumes from the host to containers run by `podman`. This is done using the path of the directory or file using the path relevant to the VM. 

For example if we wanted to mount the directory `tmp` in your user directory it would look like:

`podman run -d --name mount-test -v /mnt/Users/dalestirling/tmp:/tmp docker.io/library/busybox                                                              
`

I have found it easier to be in the directory and use: 

`podman run -d --name mount-test -v /mnt$(pwd):/tmp docker.io/library/busybox                                                              
`

You can also run the following alias to simplify it further: 

`alias podpath="echo /mnt$(pwd)"`

`podman run -d --name mount-test -v $(podpath):/tmp docker.io/library/busybox                                                              
`

I have found that this simplifies working with containers as you can work directly in the Mac space to develop solutions. 

The caveat is that the performance is not the best making this only really workable for development testing especially performance will not be accurate here. 
