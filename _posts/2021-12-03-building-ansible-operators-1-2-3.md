---
layout: single
title:  "Building Ansible Operators 1-2-3"
categories:
- OpenShift
tags:
- Containers
- OpenShift
- Operators
- Operator Framework
- Ansible
---
## Overview
In this article we will go step by step in build a Kubernetes Operator using Ansible and the Operator Framework. Operators provide ability to not only deploy an application but also manage the entire lifecycle while also baking in higher degrees of intelligence and resilience. Before we get to the Operator lets briefly revisit the ways we can deploy and manage applications in k8s. 
- Deployment Config
- Deployment Template
- Helm
- Operator

A deployment config is the simplest method but there is no way to parameterize unless you are doing it through Ansible k8s module or something else that handles that. A deployment template does provide parameterization but is only available on OpenShift and also doesn't handle packaging or management of the application itself. Helm provides parameterization and also packaging but doesn't provide application lifecycle management or allow for building extended intelligence or resilience. Operators provide the full package and provide the pattern to improve operability over time. Some suggest just use the simplest tool for the job so if you just need to deploy an app for example, don't need parameterization make a deployment config. Personally my view is that with Ansible, Operators are just about as easy as a deployment config but with so much added benefit that is always makes sense to take this approach from the beginning.
My hope and goal with this article is to maybe influence a few more Operators so let's see what we can accomplish.
## Pre-requisites
Your starting point should be an application you can deploy using a yaml deployment config. From there the next thing is to setup a development environment for building Operators using the Operator Framework.
[Operator Pre-requisites](https://docs.openshift.com/container-platform/4.8/operators/operator_sdk/ansible/osdk-ansible-quickstart.html)
Once you have operator-sdk and openshift or kubectl client you need some Ansible dependencies.
 <pre>$ sudo dnf install -y ansible</pre>
 <pre>$ sudo dnf install -y python-ansible-runner</pre>
 <pre>$ sudo dnf install -y python-ansible-runner</pre>
 <pre>$ sudo pip install ansible-runner-http</pre>
 <pre>$ sudo pip install openshift</pre>
 ## Create Operator Boiler Plate Code
 Before we can begin coding the Operator we need boiler plate code and thankfully the operator sdk does that for us.
<pre>$ operator-sdk init --plugins=ansible --domain=kubecraft.com</pre>
<pre>$ operator-sdk create api     --group cache     --version v1     --kind Kubecraft     --generate-role</pre>
## Customizing Operator
 At this point we need an application and what I always do is create a deployment yaml to ensure I have all the pieces and test deploying my application to k8s before building Operator. In this example we will use an app called [Kubekraft](https://github.com/ktenzer/kubecraftadmin). It is a fun app that connects k8s world to minecraft through a minecraft websocket server. Browse to the yaml folder and you will see the deployment.yaml. This is what we will use to build out our Operator.
Under ```roles/kubecraft/tasks``` directory we create tasks in Ansible to deploy what we had in the deployment yaml which is a deployment, service and route. In addition we added a task to get the application domain dynamically so we can build our route properly.
## Testing Operator
The operator sdk provides a simple way to test the operator locally. Once we have our tasks complete under the role we simply need to create a new project, run the operator locally and create a custom resource (CR) which executes our role and tasks we defined.

**Create project**
<pre>$ oc new-project kubecraft</pre>

**Run operator locally**
<pre>$ make install</pre>
<pre>$ make run</pre>

**Create custom resource**
In a new terminal create CR. Also note that our operator expects user input a comma separated list of namespaces to monitor. User input is parameterized via the custom resource so if you look inside you will see the namespaces parameter set and being consumed within the Ansible role.
<pre>$ oc create -f config/samples/cache_v1_kubecraft.yaml</pre>

## Building Operator
Once we have tested the operator locally we can build and publish our operator to a registry.
**Authenticate to registry**
<pre>$ sudo docker login https://registry.redhat.io</pre>

**Build operator image and push to registry**
 <pre>$ sudo make docker-build docker-push IMG=quay.io/ktenzer/kubecraft-operator:latest</pre>
 If you are using quay.io as your registry make sure to login and make the image public so it can be accessed.

## Running operator
 Now that we have the operator tested and the image built we can simply deploy it, create a CR and rule the world!
 The operator sdk makes this really easy and streamlines everything into a single command.
 <pre>$ make deploy IMG=quay.io/ktenzer/kubecraft-operator:latest</pre>

## Create operator bundle
 The operator bundle allows integration with operator lifecycle manager (olm) which provides a facility for upgrading operator seamlessly as well as integrating with operator hub. First we will generate the bundle boiler plate.
 <pre>$ make bundle</pre>
If you want to change anything, like add image you can update ```bundle/manifests``` clusterserviceversion. When your ready you will build bundle and then push it to your repository. Remember if using quay.io to make it public.
<pre>$ sudo make bundle-build BUNDLE_IMG=quay.io/ktenzer/kubecraft-operator-bundle:latest</pre>
Push bundle to registry.
<pre>$ sudo docker push quay.io/ktenzer/kubecraft-operator-bundle:latest</pre>
## Run operator bundle
Now that we have the bundle built and pushed to registry we can deploy it to a cluster.
<pre>$ operator-sdk run bundle -n kubecraft-operator quay.io/ktenzer/kubecraft-operator-bundle:latest</pre>
Once our operator bundle is running simply create a new project, a CR and watch the magic happen.
<pre>$ oc project kubecraft</pre>
<pre>$ oc create -f config/samples/cache_v1_kubecraft.yaml</pre>
<pre>$ oc get deployment
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
kubecraft   1/1     1            1           5s
</pre>
## Summary
In this article we created an operator using the operator sdk and Ansible. We saw how to take a simple deployment yaml file and turn it into a fully managed operator. Finally we created a bundle showing the integration with operator lifecycle manager and operator hub. Hopefully this article helps you get started with Ansible operators and next time you deploy an application in k8s you would consider the operator approach.

(c) 2021 Keith Tenzer




