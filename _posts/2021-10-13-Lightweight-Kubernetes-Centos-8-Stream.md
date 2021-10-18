---
layout: post
title: Deploying Kubernetes using lightweight runtimes on Centos Stream 8 
---


Thanks to the introduction of the [Kubernetes CRI](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/) there are now additional runtimes that exist for running containers within the Kubernetes orchestration ecosystem. 

In this article we will look at how we can use a lightweight runtime layer to create the smaller resource overheads when running container. 

The tools that we are going to utilise to meet this objective is the [Cri-o CRI](https://cri-o.io/) and the [Crun OCI runtime](https://github.com/containers/crun#crun). 

Cri-o is a container runtime interface that is purpose built to run containers in Kubernetes and does not contain additional features like other CRI implemantations such as `containerd`. It is able to do this by hading off many tasks to [libpod and other tools from the Container Tools project](https://www.capitalone.com/tech/cloud/container-runtime/). 

Crun is a lightweight and performant OCI runtime that is developed in C. This OCI implementation is part of the [Containers project](https://github.com/containers) like `podman`, `buildah` and others. This implementation is able to use tighter resource limits than `runc` as illustrated in the example below:  

```bash
# podman --runtime /usr/bin/runc run --rm --memory 4M fedora echo it works

Error: container_linux.go:346: starting container process caused "process_linux.go:327: getting pipe fds for pid 13859 caused \"readlink /proc/13859/fd/0: no such file or directory\"": OCI runtime command not found error

# podman --runtime /usr/bin/crun run --rm --memory 4M fedora echo it works

it works
```

In this example we see that when runnung the same `echo` command withing the fedora container image the container can execute with only 4 megabytes of memory using `crun` but raises an error when executed using `runc`.

This Kubernetes topology can be deployed to a Centos 8 Stream relatively quickly using RPM packages making ongoing management of the Kubernetes platform more manageable as new versions are released. 

The assumption of this guide is that a clean install of Centos 8 Stream has been deployed and updated. During my install I enabled the Container Tools AppStream module and installed these tools through the installer GUI.   

Once this is done we can move on with preparing the Centos OS to run the Kubernetes stack. 

First we disable the swap partition to manage the performance of the platform if it become oversubscribed for CPU or Memory. 

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Next we need to ensure that the correct modules are present in the kernel. These modules ensure that Centos Kernel is able to manage traffic as required by Kubernetes. 

```bash
sudo modprobe br_netfilter
sudo modprobe overlay
```

Then these modules need to be included into the Kernel at boot. 

```bash
sudo touch /etc/modprobe.d/br_netfilter.conf
sudo touch sudo touch /etc/modprobe.d/overlay.conf
```

Many of the tutorials that I read while I was looking into this called for the local firewalls on the host to be disabled. This is not the case with a few simple commands you can add firewalld rules. 
So that traffic is able to flow on the Service and Pod networks we need to enable Masquerade in firewalld. 

```bash
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
```

While working on the firewall we can add the required ports for the cluster to operate with the service layer. Make sure that you do a review of your existing firewalld configuration to ensure that this will not impact the current state of your host. 
These rules will enable external nodes to connect to cluster services and allow for internal connectivity to the service layer.

```bash
sudo firewall-cmd --zone=public --permanent --add-port={6443,2379,2380,10250,10251,10252}/tcp
sudo firewall-cmd --zone=public --permanent --add-rich-rule 'rule family=ipv4 source address=172.17.0.0/16 accept'
sudo firewall-cmd --reload
```

Next we use `sysctl` to enable Kernel settings to allow for packet forwarding and bridged packets to traverse iptables. Both of these are require for the Kubernetes network interface (CNI) to operate. The settings are stored in a public GitHub Gist to make the setup simpler. 

{% gist 316eae008bb123b783f90cb5ef8633b0 99-kubernetes-cri.conf %}

```bash
sudo curl -o /etc/sysctl.d/99-kubernetes-cri.conf https://gist.githubusercontent.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0/raw/56b34f93fb0528aa6818141fbd3e0f5f36db39b1/99-kubernetes-cri.conf
sudo sysctl --system
```

Now the host is prepared to deploy Kubernetes. To do this we first need to deploy our container runtime and CRI.

To deploy cri-o we will be using the EPEL repositories for Centos stream 8, I chose to take this path over the recommended cri-o repository in the Docs as this repository of the time of writing did not work, additionally the packages in the EPEL repository are built against Centos. 
First we need to enable the repository using the package manager. Cri-o is distributed as an AppStream module so this will also need to be enabled so packages can be installed. 

```bash
dnf install epel-release
dnf modules enable cri-o
```

Now we can install Cri-o using package management allowing easy updates as new versions are released.

```bash
sudo dnf install cri-o
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

Now that we have the CRI installed we need to configure Cri-o to set the cgroups driver. 

{% gist 316eae008bb123b783f90cb5ef8633b0 02-cgroup-manager.conf %}

```bash
sudo mkdir -p /etc/crio/crio.conf.d
sudo curl -o /etc/crio/crio.conf.d/02-cgroup-manager.conf https://gist.githubusercontent.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0/raw/a0881b89a86420b874d6716205d669c0dbdd63cd/02-cgroup-manager.conf
```

And restart the service to have Cri-o accept the new configuration that was created.

```bash
sudo systemctl restart crio
```

Currently Cri-o is running runc as this is the default runtime that is installed. Runc is the default runtime installed when deploying components from the Container Tools project in Centos Stream. 

To transition to Crun to complete our lightweight tool chain we can simply install and configure Cri-o to use it.

```bash
sudo dnf install crun
```

Adding the config to Cri-o can be done using adding to config to crio.conf.d directory we created earlier. After this config is added Cri-o will need a restart. 

{% gist 316eae008bb123b783f90cb5ef8633b0 03-runtime-crun.conf %}

```bash
sudo curl -o /etc/crio/crio.conf.d/03-runtime-crun.conf https://gist.githubusercontent.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0/raw/8b019a4e3c607a09d4d80e5989ac56657a72fd08/03-runtime-crun.conf
sudo systemctl restart crio 
```

Now that we have a runtime we can install Kubernetes to manage containers using the runtime we installed. The first step in getting this done is to add the kubernetes repository that holds the vanilla images. 

{% gist 316eae008bb123b783f90cb5ef8633b0 kubernetes.repo %}

```bash
sudo curl -o /etc/yum.repos.d/kubernetes.repo https://gist.githubusercontent.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0/raw/56b34f93fb0528aa6818141fbd3e0f5f36db39b1/kubernetes.repo
```

This repository definition utilises the `exclude` definition to ensure that the Kubernetes RPMS are not updated without your knowledge. This is done so that updates to Kubernetes can be planned and impacts to workloads minimised.

To install the kubernetes RPMs we need to override the exclude with the `--disableexcludes` option for the kubernetes repository.

```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Now that we have the service in place we need to enable and restart the `kublet` service. I found that for some reason that I could not work out it needed this.

```bash
sudo systemctl enable --now kubelet
sudo systemctl restart kubelet
```

Once `kublet` is running we can use `kubeadm` to pull all of the cluster images so when we initialise the cluster it can deploy the correct components. 

```bash
sudo kubeadm config images pull
```

This step can take some time depending on connectivity. 

Once you have pulled the images you can stand-up a Kubernetes cluster using `kubeadm` to initialise the new cluster. 

{% gist 316eae008bb123b783f90cb5ef8633b0 kubeadm-config.yaml %}

```bash
curl -o /tmp/kubeadm-config.yaml https://gist.githubusercontent.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0/raw/08a76210c9cc5e85e005bf812d731a4fcec24932/kubeadm-config.yaml
sudo kubeadm init --config /tmp/kubeadm-config.yaml
```

This will deploy all of the containers that make up the control plane and generate the required keys to interact with the API server. 

Now we have a functioning Kubernetes cluster and you should be able to use `kubectl` to interact with the cluster. As we are building a single node cluster we need to relabel the node to be a worker as well as a master. 

```bash
kubectl label node localhost.localdomain node-role.kubernetes.io/worker=worker
```

Now you should have a working cluster that you can interact with using `kubectl` 

All of the config files are available in the following [GitHub Gist](https://gist.github.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0).

####Update: CGroups error. 
Saw the following error in `journalctl` while troubleshooting a deployment issue. 

```bash
"Failed to find cgroups of kubelet" err="cpu and memory cgroup hierarchy not unified. cpu: /, memory: /system.slice/kubelet.service"
```

This can be corrected by adding some additional cgroups configuration. 

{% gist 316eae008bb123b783f90cb5ef8633b0 kubeadm-config.yaml %}

```bash
sudo mkdir -p /etc/systemd/system/kubelet.service.d
sudo curl -o /etc/systemd/system/kubelet.service.d/11-cgroups.conf https://gist.githubusercontent.com/dalethestirling/316eae008bb123b783f90cb5ef8633b0/raw/1e1ed6922c7f01db971ab21508ad28aba27e5933/11-cgroups.conf
systemctl daemon-reload && systemctl restart kubelet
```
This will add the required options to enable the cpu and memory interfaces. 

####Update: Taint on node.
As the cluster is a single node it contains all of the roles. Master hosts are tainted for `NoSchedule` by default. This means that non Pods are able to be deployed to the cluster. This can be removed with the following:

```bash
kubectl taint node localhost.localdomain node-role.kubernetes.io/master:NoSchedule-
```
My single node cluster proceeded to reschedule everything and be a mess for several minuites, but sorted itself out after that. 
