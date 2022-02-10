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
In this article we will go step by step to build a Kubernetes Operator using Ansible and the Operator Framework. Operators provide the ability to not only deploy an application but also manage the entire lifecycle while also baking in higher degrees of intelligence as well as resilience. Before we get to the Operator lets briefly revisit the ways we can deploy and manage applications in k8s. 
- Deployment
- Deployment Template
- Helm
- Operator

A k8s deployment is the simplest method but there is no way to parameterize unless you are doing it through Ansible k8s module or something else that handles that. A deployment template does provide parameterization but is only available on OpenShift and also doesn't handle packaging or management of the application itself. Helm provides parameterization and also packaging but doesn't provide application lifecycle management or allow for building extended intelligence for added resilience. Operators provide the full package and provide the pattern to improve operability over time. Some suggest just using the simplest tool for the job. If you just need to deploy an app for example and don't need parameterization, use a k8s deployment. Personally my view is that with Ansible, Operators are just about as easy as a k8s deployment but with so much added benefit that the Operator approach always makes sense. My hope and goal with this article is to maybe influence a few more Operators and show that it isn't really any additional work.

## Pre-requisites
Your starting point should be an application you can deploy using a k8s deployment or deployment config. From there the next thing is to setup a development environment for building Operators using the Operator Framework and Ansible.
[Operator Pre-requisites](https://docs.openshift.com/container-platform/4.8/operators/operator_sdk/ansible/osdk-ansible-quickstart.html)
Once you have operator-sdk and openshift or kubectl client you need some Ansible dependencies.
 <pre>$ sudo dnf install -y ansible</pre>
 <pre>$ sudo dnf install -y python-ansible-runner</pre>
 <pre>$ sudo dnf install -y python-ansible-runner</pre>
 <pre>$ sudo pip install ansible-runner-http</pre>
 <pre>$ sudo pip install openshift</pre>
 <pre>$ ansible-galaxy collection install kubernetes.core </pre>
 <pre>$ ansible-galaxy collection install operator_sdk.util </pre>


## Create Operator Scaffolding
 Before we can begin coding the Operator we need boiler plate code and thankfully the operator sdk does all that.
<pre>$ operator-sdk init --plugins=ansible --domain=kubecraft.com</pre>
<pre>$ operator-sdk create api     --group cache     --version v1     --kind Kubecraft     --generate-role</pre>

## Customizing Operator
At this point we need an application. My approach is to first create a k8s deployment and test deploying my application before building the Operator. In this example we will use an app called [Kubekraft](https://github.com/ktenzer/kubecraftadmin). It is a fun app that connects k8s world to minecraft through a minecraft websocket server written in Go. Browse to the yaml folder and you will see the k8s deployment.yaml. This is what we will use to build out our Operator.
Under ```roles/kubecraft/tasks``` directory we create tasks in Ansible to deploy what we had in the k8s deployment yaml which is a deployment, service and route. In addition we added a task to get the application domain dynamically so we can build our route properly. This also demonstrates how to query and use other k8s resources in our Ansible code.

In addition, if your operator creates k8s resources you need to ensure proper permissions. Under the ```config/rbac``` directory you can add permissions to the role.yaml. For this operator I added services and routes so that those resources  can be created by the Operator.

## Testing Operator
The operator sdk provides a simple way to test the operator locally. Once we have our tasks complete under the ```role/project/tasks``` directory, we simply need to create a new project, run the operator locally and create a custom resource (CR) which executes our role and tasks we defined.

**Create project**
<pre>$ oc new-project kubecraft</pre>

**Run operator locally**
Run these commands from the root directory of your operator project where the Makefile resides.
<pre>$ make install</pre>
<pre>$ make run</pre>

**Create custom resource**
In a new terminal create CR. Also note that our operator expects as user input, a comma separated list of namespaces to monitor. User input is parameterized via the custom resource so if you look inside you will see the namespaces parameter set and being consumed within the Ansible role.
<pre>$ oc create -f config/samples/cache_v1_kubecraft.yaml</pre>

## Building Operator
Once we have tested the operator locally we can build and publish our operator to a registry.
**Authenticate to registry**
<pre>$ sudo docker login https://registry.redhat.io</pre>

**Build operator image and push to registry**
 <pre>$ sudo make docker-build docker-push IMG=quay.io/ktenzer/kubecraft-operator:latest</pre>
 If you are using quay.io as your registry make sure to login and make the image is public so it can be accessed.

## Running Operator
 Now that we have the operator tested and the image built we can simply deploy it, create a CR and rule the world!
 The operator sdk makes this really easy and streamlines everything into a single command.
 <pre>$ make deploy IMG=quay.io/ktenzer/kubecraft-operator:latest</pre>

 BY default the operator will be installed into a project ```operatorName-system``` however you can change that by updating the project name in the PROJECT file under the root of the operator project. In this case we changed it to kubecraft-operator.
 
 We can remove the operator also using make.
 <pre>$ make undeploy</pre>

## Create Operator Bundle
 The operator bundle allows integration with operator lifecycle manager (olm) which provides a facility for upgrading operator seamlessly as well as integrating with operator hub. First we will generate the bundle boiler plate.
 <pre>$ make bundle</pre>
If you want to change anything, like add image you can update ```bundle/manifests``` clusterserviceversion. When your ready you will build bundle and then push it to your repository. Remember if using quay.io to make the image public.
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
In this article we created an operator using the operator sdk and Ansible. We saw how to take a simple k8s deployment and turn it into a fully managed operator. Finally we created a bundle showing the integration with operator lifecycle manager and operator hub. Hopefully this article helps you get started with Ansible operators and next time you deploy an application in k8s you would consider the operator approach.

(c) 2021 Keith Tenzer




