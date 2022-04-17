---
layout: post
title: Deploying pi-hole-docker on Kubernetes
---

The internet can be a scary place, especially when you have kids. 

I have tried several methods to control and filter the use of technology to keep my kids safe and give them the freedom to explore and be creative. 

Recently, I have been exploring how DNS can be managed to assist in managing content and removing trackers that misdirect and influence consumers. 

[Pi-hole](https://pi-hole.net/) is an on-premise DNS server that provides DNS filtering and black-hole lists to remove advertising. 
Pi-hole can be deployed on low power hardware like a Raspberry-pi using Linux via the installer or [Docker container](https://github.com/pi-hole/docker-pi-hole).

I didn't have a host running docker or a spare Pi laying about, but what I did have was a Kubernetes host that was built as part of this post. So I thought why not!

Deploying to Kubernetes gives you access to an additional layer of orchestration that can help with the availability of the service in the event it crashes. 

Now you could just translate the provided `docker-compose.yaml` that is supplied in the [docker containers GitHub repository](https://github.com/pi-hole/docker-pi-hole) to a Kubernetes Pod resource like this. 

{% gist ad5af29ada166489867d1cd2555b7f19 pi-hole-pod.yaml %} 

This gives you the equivalent of running the container in Docker, but we can take this further by defining a few more resources to get the most out of Kubernetes.
 
The next evolution of getting Pi-hole onto Kubernetes is to describe your desired state using a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) resource. Deployments provide the ability to be able to roll out new versions or roll back to previous iterations. 

They also provide the ability to scale your workload to more than one instance of a Pod. This coupled with anti-affinity deployment models could be interesting for DNS servers.

{% gist bc9bc22f8acb855187dec0cad17f7f58 deployment.yaml %}

A few key things to point out. You will notice that the configuration is managed within the Deployment rather than in a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) and referenced in. This is to leverage the rollout capabilities of the Deployment resource, Deployment rollouts are not triggered if you update the ConfigMap.

The other thing is the ports, these are privileged ports. Normally you need to use ports that are located in the range supplied by your `kublet` config. This can be overridden as the Deployment contains both the privileged: true and `hostNetwork: true` options set in the resource definition. 

To deploy to Kubernetes you can just submit the resource definition to the server using `kubectl apply -f <filename>`. This will give you a running instance. 

Next, you need to expose the Pi-home Pod outside Kubernetes so devices can access DNS services. For my home deployment, I have a single node so the use of an ingress solution is not required and I use a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) defining Ports on the host to expose the solution. 

{% gist bc9bc22f8acb855187dec0cad17f7f58 service.yaml %}

The service uses the same privileged ports as DNS services are known to exist on port 53. You will also notice that there are 2 separate Service definitions as you need to define one for each protocol type. 

These can be applied using `kubectl` in the same way that the Deployment was previously.

This will now give you a running instance of Pi-hole that you can update your DHCP server to point to. 
I have also configured the instance to forward its requests to [OpenDNS](https://www.opendns.com/) for an additional layer of security and filtering. 

I am maintaining the [deployment and service YAML files on GitHub here](https://github.com/dalethestirling/pi-hole-kube).

**NOTE:** 16/4 - I could not get any of the 2022.04 releases to deploy due to the way the start script works with assigning capabilities.