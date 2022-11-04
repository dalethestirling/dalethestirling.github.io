---
layout: post
title: Why it is better to promote your containers
tags: containers ci-cd devops kubernetes
---

There are 2 schools of thought on how to track container artifacts that have made their way through your quality gates and into production where they can be consumed by your end users. 

One approach is to use tagging to manage all of your container versions in a single container registry that contains all versions of a container image that has been created. This leaves you with a mix of failed development and test candidate containers residing in the same registry as the images that have progressed to production through these quality gates. 

<img alt="Single registry diagram" src="{{ site.url }}/images/single-registry-diagram.png" width="550">

The diagram above illustrates a simplified depiction of this approach. In this diagram, there are 3 versions of the container, each container is tagged with its semantic version, and the currently deployed container to production also has the tag `prod`. The newest version (1.0.3) is in non-production but tagged `latest` as it is the most recent version added to the registry.

What this approach can't illustrate is if the container tagged `1.0.1` was a container that was deployed to production or not. This can lead to challenges that are discussed below. 

The other approach is to have 2 container registries to demarcate the containers that have made it through the SDLC's quality gates and into the production system. This provides segmentation from the other failed deployment candidate containers. 

 <img alt="Multiple registry diagram" src="{{ site.url }}/images/multiple-registry-diagram.png" width="550">

As illustrated in the diagram above contains the same 3 versions committed to container registries as the scenario from earlier. In this topology, we have removed the need for environment tags and can rely on the inbuilt `latest` tag to denote the newest version pushed into a registry. Also, the semantic version number is retained to allow reconciliation of the container to a pipeline build activity as well as version pinning in its associated deployment assets. 

### Why I prefer to promote rather than tag
The goal of building a successful platform is to provide easy-to-consume capabilities for users of the platform. This can at times lead to having to absorb complexity and effort in its implementation to abstract this from the user. The promotion model for containers is a great example of this.

While there may be more to set up in a promotion topology as there are multiple container repositories and additional pipeline tasks, the benefit of this capability justifies the setup effort.

From my experiences using both models here are some examples of where this benefit to users exists.

#### A gate to progress to production
The most obvious thing that this approach offers is a step where a container needs to be physically pulled from one registry and pushed to another. This step can be as manual or as automated as your risk appetite will allow. 

This presents a place for approvers to accept a container artifact into Production. This approval gate should have documented what is required for a container to be promoted to Production, these may include things in addition to quality code like documentation, test acceptance or addition to the CMDB. 

Approvers should be from the actors that will have to manage the code for its life in the Production environment, these may be the Product Owner and/or Operations Team.

#### Mitigate rollback risk
Using a Production only repository means that all images that are in the registry are "Production Ready" deployments and have passed through all of the quality gates defined in the SDLC to qualify for promotion. 

This means that where there is a requirement to roll a containerised component back, whatever the reason all of the images available a good to be consumed and all that needs to be considered is the compatibility of the component within the system. 

This reduces the risk of a version that has failed quality gates and has not been eligible for production, being deployed due to error. This also is true for the use of default tags such as `latest` that may be automatically added to containers.

#### It provides a defined security scope
The scanning of containers for security reporting can be a confusing activity as the newest container image may not be the container deployed in Production. 

This can lead to artefacts that did not meet quality gates or abandoned images may be aggregated into scanning as they are generally done at the repository level. 

Having a Production only repository means that the reporting can be far more focused, with only container images that are eligible to run in Production being captured. This allows less effort in filtering reports to present a more accurate representation of risk in your Production components. 

#### Demarcation of responsibility
As spoken about above a promotion model has a clear demarcation between container images that are in phases of development and those that are ready to be deployed and managed in Production. 

This makes compliance with Secure Software Development practices and controls simpler as it is baked into the architecture of the platform.

This topology also aligns with the guidelines of [Software Development](https://www.cyber.gov.au/acsc/view-all-content/advice/guidelines-software-development) from the Information Security Manual (ISM) where `Development, testing and production environments are segregated.` (ISM-0400) and the `Unauthorised access to the authoritative source for software is prevented.` (ISM-1422).

This is why I choose in my platform deployments to advocate the adoption of a promotion process and multiple registries approach. It may take more effort to get the process off the ground but once it is in place it simplifies the lifecycle of containers and their management. 

I have seen this solution work with both hosted container registries, container registries as a service (Cloud Hyperscaler or 3rd party) or a hybrid of the two. 

I am keen to hear how this approach has worked for others, so reach out on [LinkedIn](https://www.linkedin.com/in/dalethestirling/) or [Twitter](https://twitter.com/dalethestirling) and let me know. 
