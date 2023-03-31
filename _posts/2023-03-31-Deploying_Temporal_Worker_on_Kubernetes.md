--- 
layout: single
title:  "Deploying Temporal Workers on Kubernetes"
categories:
- Temporal
tags:
- Temporal Cloud
- Temporal Worker
- Kubernetes
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will show how to bootstrap a Temporal worker on Kubernetes. The Temporal worker is a service built using the Temporal SDK that executes workflows and activities. A Temporal worker polls the Temporal cloud or server and communicates over gRPC using mTLS. In addition to mTLS certificates, a Temporal worker also needs the Temporal cloud host endpoint and namespace. If the Temporal worker will interact with any other services or databases it may require additional authorization keys or passwords. 

![Temporal](/assets/2023-03-31/temporal.png)

In Kubernetes, to ensure things are dynamic, all of these settings can be injected into the Temporal worker pods via environment parameters. For sensitive information like mTLS certificates and any authorization keys or passwords, secrets should be used. Those secrets can exist in Kubernetes or be managed by an external secrets store. One can even store the secrets on a encrypted volume and mount them inside the pod/container.

## Use Environment Variables in Temporal Worker
The first step is to ensure the worker configuration is dynamic. This example in Go shows how to do that with environment variables.

<pre>
...
clientOptions := client.Options{
	HostPort:  os.Getenv("TEMPORAL_HOST_URL"),
	Namespace: os.Getenv("TEMPORAL_NAMESPACE"),
}

cert, err := tls.LoadX509KeyPair(os.Getenv("TEMPORAL_MTLS_TLS_CERT"), os.Getenv("TEMPORAL_MTLS_TLS_KEY"))
if err != nil {
	log.Fatalln("Unable to load certs", err)
}

clientOptions.ConnectionOptions = client.ConnectionOptions{
	TLS: &tls.Config{
		Certificates: []tls.Certificate{cert},
	},
}
...
</pre>    

## Build a Docker Image
Next we need to build a docker image for our Temporal worker. This will vary, depending on the Temporal SDK programming language. In this example, a Dockerfile for a Temporal worker written in Go is shown.

<pre>
$ vi Dockerfile
FROM docker.io/library/golang:1.18-alpine
RUN mkdir -p /app/bin
WORKDIR /app
ENV GOBIN=/app/bin
COPY . .

RUN go install . ./worker

FROM docker.io/alpine:latest
LABEL ios.k8s.display-name="backup-worker" \
    maintainer="Keith Tenzer <keith.tenzer@temporal.io>"

RUN mkdir -p /app/bin
WORKDIR /app/bin
COPY --from=0 /app/bin/worker /app/bin
CMD ["/app/bin/worker" ]
</pre>

Build docker image.
<pre>
$ docker build -t ktenzer/temporal-worker:v1.0 .
</pre>

Push docker image to docker.io.
<pre>
$ docker push ktenzer/temporal-worker:v1.0
</pre>

## Create Secret for mTLS Certificates
Kubernetes has a few different secret types, one of them is tls. When a secret is created it is stored in base64. One can use kubectl to create the secret, or manually convert certificates to base64 and then add those secrets to a yaml file.

The tls.crt is your PEM or public portion of the certificate that you also upload to your Temporal cloud namespace. The tls.key is the private key associated with the certificate. The client of course needs both.

Using kubectl.
<pre>
$ kubectl create secret tls my-tls-secret --key /path-to/ca.key --cert /path/to/ca.pem -n temporal-worker
</pre>

Using yaml.
<pre>
$ vi tls-secret.yaml
apiVersion: v1
data:
  tsl.crt: TLS Certificate Base64
  tls.key: TLS Certificate Base64
kind: Secret
metadata:
type: kubernetes.io/tls
</pre>

<pre>
$ kubectl create -f tls-secret.yaml
</pre>

## Create Secrets for other Services
It is often the case, that a Temporal worker also communicate with other services, which may require some authorization. For example, if we wanted to interact with ChatGPT, an API key is needed. That API key should also be stored as a secret. In this case we would use a generic secret.

Using kubectl.
<pre>
$ kubectl create secret generic chatgpt-key --from-literal=KEY=API key -n temporal-worker
</pre>

Using yaml.
<pre>
$ vi chatgpt-secret.yaml
apiVersion: v1
data:
  KEY: API KEY Base64
kind: Secret
metadata:
  name: chatgpt-key
type: Opaque
</pre>

<pre>
$ kubectl create -f chatgpt-secret.yaml -n temporal-worker
</pre>

## Creating Kubernetes Deployment
A deployment in kubernetes manages the application in a dynamic, ephemeral way. It controls replicas, environment, probes, image and much more. For a Temporal worker there are a few things to consider. We will at minimum want to set the image or container, deployment strategy, resource limits, liveliness/readiness probes and of course, inject environment parameters.

### Image
This is straightforward and is just the container image version that should be run.
<pre>
image: ktenzer/temporal-worker:v1.0
</pre>

### Deployment Strategy
The deployment strategy defines how changes or updates are handled. We can do rolling updates, blue/green and even a canary type of deployment. Here a rolling deployment is configured, 25% of the pods will be updated before moving to next 25%.

<pre>
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
  type: RollingUpdate
</pre>

### Liveliness and Readiness Probes
Liveliness and readiness probes are really important. This is how kubernetes knows if the Temporal worker is properly functioning. Normally you would have a /status endpoint to test over HTTP/gRPC, but in the case of a Temporal worker, there is no endpoint, no service is exposed, since the Temporal worker polls. As such we have to be a bit more creative. Thankfully kubernetes allows us to exec into pods where we can run a command. Ideally, the worker should output something to a file and then have the probe check the file, via exec with a regex. In this case we are just doing liveliness/readiness probes using ls inside pod. 

<pre>
readinessProbe:
  exec:
    command:
    - ls
    - /
  failureThreshold: 3
  initialDelaySeconds: 5
  periodSeconds: 5
  successThreshold: 1
  timeoutSeconds: 1
livenessProbe:
  exec:
    command:
    - ls
    - /
  failureThreshold: 3
  initialDelaySeconds: 5
  periodSeconds: 5
  successThreshold: 1
  timeoutSeconds: 1
</pre>

### Resource Limits
In kubernetes there are resource requests and limits. Requests is initially what a pod gets when it is started and what it will get additionally, when it requires more resources. The limit is the maximum amount of a resource that a pod can have.

<pre>
resources:
  limits:
    cpu: 500m
    memory: 500Mi
  requests:
    cpu: 100m
    memory: 100Mi
</pre>    

### Environment
The environment is where we can inject any secrets and also set other important settings for proper operation of a Temporal worker. For the secrets we need to first mount them into the pod. 

First we define a volume and what secret it will mount.
<pre>
volumes:
- name: certs
  secret:
    defaultMode: 420
    secretName: tls-secret
</pre>

Next we mount the volume inside the container.
<pre>
volumeMounts:
- mountPath: /etc/certs
    name: certs
</pre>          

Finally, we inject the secret into an environment variable.
<pre>
- name: TEMPORAL_MTLS_TLS_CERT
  value: /etc/certs/tls.crt
- name: TEMPORAL_MTLS_TLS_KEY
  value: /etc/certs/tls.key
</pre>

For the ChatGPT secret, a generic secret is used which can be injected directly.
<pre>
- name: CHATGPT_API_KEY
  valueFrom:
    secretKeyRef:
        key: KEY
        name: chatgpt-key
</pre>

Lastly, we also need to set the Temporal cloud host endpoint and namespace.
<pre>
- name: TEMPORAL_HOST_URL
  value: namespace.accountId.tmprl.cloud:7233
- name: TEMPORAL_NAMESPACE
  value: namespace.accountId
</pre>

### Brining it all together
Below is the entire deployment yaml for this example.

<pre>
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/build: "1"
    app.kubernetes.io/component: worker
    app.kubernetes.io/name: temporal-worker
    app.kubernetes.io/version: v1.1
  name: temporal
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: worker
      app.kubernetes.io/name: temporal-worker
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/build: "1"
        app.kubernetes.io/component: worker
        app.kubernetes.io/name: temporal-worker
        app.kubernetes.io/version: v1.0
    spec:
      containers:
      - env:
        - name: CHATGPT_API_KEY
          valueFrom:
            secretKeyRef:
              key: KEY
              name: chatgpt-key
        - name: TEMPORAL_HOST_URL
          value: namespace.accountId.tmprl.cloud:7233
        - name: TEMPORAL_NAMESPACE
          value: namespace.accountId
        - name: TEMPORAL_MTLS_TLS_CERT
          value: /etc/certs/tls.crt
        - name: TEMPORAL_MTLS_TLS_KEY
          value: /etc/certs/tls.key
        image: ktenzer/temporal-worker:v1.0
        imagePullPolicy: Always
        readinessProbe:
          exec:
            command:
            - ls
            - /
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1        
        livenessProbe:
          exec:
            command:
            - ls
            - /
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        name: temporal-worker
        resources:
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 100Mi
        securityContext:
          allowPrivilegeEscalation: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/certs
          name: certs
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: certs
        secret:
          defaultMode: 420
          secretName: tls-secret
</pre>

## Summary
In this article we showed how to bootstrap a Temporal worker in kubernetes. Creating secrets and injecting them into a kubernetes deployment allows for a dynamic, secure way to manage Temporal workers in kubernetes. Finally, we walked through the various steps in operationalizing a Temporal worker on kubernetes: dynamic worker configuration, building a docker image, creating secrets and of course wiring it all together in a kubernetes deployment.

(c) 2023 Keith Tenzer




