---
layout: post
title: Deploying Homeassistant on Kubernetes
---

When we moved to the new house, one of the things we did was step up our home automation from Google home/Action blocks to Home Assistant. 

To do this I used a Raspberry Pi 4, this was a great platform for Home Assistant and a very complete solution.

That was until I woke up to no Home Assistant dashboards and my Raspberry Pi flashing an error code on the ACT light informing me that the SDRAM was now dead on the board. This may have been my fault for not having a case for it.

<iframe width="560" height="315" src="https://www.youtube.com/embed/mzpBz2FWyIA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The video is from `CarpeDiemOther`'s channel and is an example of my issue.

As many will be aware due to the global shortage of semiconductors picking up a replacement is getting close to impossible. So I decided to use the hardware I had in the house already. This is so I can ride out the shortage till I can get myself a Home Assistant Yellow. 

What better platform to utilise for this than my Kubernetes server that I built (and blogged about) a while ago? To improve things, there is already a container build maintained by the Homeassistant project. 

So when looking to run the container version of Homeassistant you cannot run [Add-ons](https://www.home-assistant.io/addons) so you may need to run things independently of Homeassistant. 

A well-maintained `helm` chart for Home Assistant is part of the [k8s-at-home](https://k8s-at-home.com) project. This is a great way to manage the lifecycle of Home Assistant as upgrades will now be done through the deployment of containers. 

Saying that I am looking to use ArgoCD and GitOps to manage my Kubernetes assets with native YAML unfortunately, these are not as well maintained. I have put the assets that I am using in [GitHub](https://github.com/dalethestirling/homeassistant-kube) that deploy my Home Assistant. 

To deploy Home Assistant on Kubernetes you need to have a place to store the state and config of your home automation when you do upgrades etc. This can be through either NFS storage leor as I did on the local host of the Kubernetes host. 

Deployment artefacts for the workload are straightforward using this approach. First, you need a namespace so your container has a home to be deployed to. 

{% gist 4afce63eb152c8763bb7b40260bcac39 namespace.yaml %}

Now that you have somewhere to deploy a container to you need to deploy the artefact. You could use a `Pod` resource here to deploy the container, but this requires the deletion and recreation of this asset with every time you want to change the Pod Spec. Instead I will be using a `Deployment` resource so that I am able to minimise the downtime with [Kubernetes rolling deployments](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/). 

{% gist 4afce63eb152c8763bb7b40260bcac39 deployment.yaml %}

This version currently uses the `stable` tag meaning that with every restart the latest stable build will be used. I have also set `imagePullPolicy: Always` to ensure that Kubernetes will download the latest version if it exists. 

The deployment also defines a volume mount for the `/config` directory. This ensures that none of the config of Homeassistant is lost between `Pod` restarts or deployments. As my environment is a single node, I am using local storage on the host. 


```
      volumes:
        - name: config
          hostPath:
            path: /opt/volumes/homeassistant/config
        - name: localtime
          hostPath:
            path: /etc/localtime

``` 

You can use whatever type of volume that can be mounted within the container. As you can see I also mount `/etc/localtime` from my server to ensure that Homeassistant is using the correct time zone. Alternatively, you can also define a separate timezone file if your server and container use different timezones.

Now that the `Pod` is deployed to the cluster we need to expose the service to consumers. As my Kubernetes cluster is a single node I do not have to worry about conflicts and do not run an ingress controller like `nginx` and manually assign my services ports via a `nodePort`.

{% gist 4afce63eb152c8763bb7b40260bcac39 service.yaml %}

