--- 
layout: single
title:  "Temporal Cloud Certificate Rotation"
categories:
- Temporal
tags:
- Certificates
- mTLS
- Kubernetes
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will look at how to rotate certificates in Temporal cloud. There are two possible certificate rotation scenarios.
- Scenario 1: Rotate Cloud CA Certificate
- Scenario 2: Rotate Client Leaf Certificate

In Scenario 1, we need to not only rotate the cloud CA certificate but also rotate any client leaf certificates generated using the CA. In Scenario 2, only new client leaf certificates need to be rotated. 

Usually, it makes sense to rotate the client leaf certificates more frequently, maybe monthly, weekly, or even daily. The cloud CA certificate can be rotated more infrequently, for example, yearly. Since each scenario is a bit different, you will want to obviously automate both. 

## Scenario 1: Rotate Cloud CA Certificate
In this scenario, the cloud CA certificate is expiring and needs to be replaced. Temporal will notify customers of such events, 10 days prior to certificate expiration via email. You can setup an email filter, or just be on lookout for emails from ```noreply@temporal.io```, or with subject ```[Temporal Cloud] Action Required: CA Certificate Expiring Soon```.

### Generate New CA Certificate
There are of course many options for generating self-signed certificates. Below I will cover certstrap and tcld. Certstrap allows for more flexibility and options, however, tcld is the standard cli from Temporal for managing Temporal cloud namespaces.

#### Certstrap
Certstrap will place certificates by default under a folder called ```out```.

Create a new CA certificate.
```bash 
$ certstrap init --common-name "Booking"
```

```bash
$ certstrap request-cert --common-name temporal-booking.sdvdw
```

#### TCLD
Generate new CA certificate.
```bash
$ tcld gen ca --org Temporal -d 365d --ca-cert /home/ktenzer/temporal/certs/Booking.crt --ca-key /home/ktenzer/temporal/certs/Booking.key
```
### Upload CA Certificate
Authenticate to Temporal Cloud
```bash
$ tcld l
```

Upload new CA certificate to Temporal Cloud namespace
```bash
$ tcld n ca add --namespace temporal-booking.sdvdw -f out/Booking.crt
```

Add any certificate filters. In this case common-name.
```bash
$ tcld n cf add --namespace temporal-booking.sdvdw --input "{ \"filters\": [ { \"commonName\": \"temporal-booking.sdvdw\" } ] }"
```

## Scenario 2: Rotate Client Leaf Certificate
Rotating client leaf certificates involves generating new certificate, testing and restarting workers with the new certificate.

### Certstrap
Sign certificate which will generate leaf certificate.
```bash
$ certstrap sign temporal-booking.sdvdw --CA "Booking"
```

### TCLD
Generate new client leaf certificate.
```bash
$ tcld gen leaf --org Temporal --common-name temporal-booking.sdvdw -d 364d --ca-cert /home/ktenzer/temporal/certs/Booking.crt --ca-key /home/ktenzer/temporal/certs/Booking.key --cert /home/ktenzer/temporal/certs/temporal-booking.sdvdw.crt --key /home/ktenzer/temporal/certs/temporal-booking.sdvdw.key
``` 

### Test new CA Certificate
There are a lot of ways to test the new client leaf certificate. I recommend using temporal cli, and listing workflows using the newly generated certificate.
```bash
$ temporal workflow list --address temporal-booking.sdvdw.tmprl.cloud:7233 --namespace temporal-booking.sdvdw --tls-cert-path /path/to/temporal-booking.sdvdw.crt --tls-key-path /path/to/temporal-booking.sdvdw.key
```
Even if you don't have any workflows, if there is a problem with the certificate, you will definitely see an error.

### Update Workers
Now that we have tested, we can safely update our worker configuration. In Kubernetes, certificates are stored inside secrets, and injected into a worker pod.

Update or create new secret, adding the new client leaf key and certificate.
```bash 
$ kubectl create secret tls temporal-booking-tls --key /path/to/temporal-booking.sdvdw.key --cert /path/to/temporal-booking.sdvdw.crt -n temporal-trivia
``` 

Restart workers by scaling deployment.
```bash
kubectl scale deployment temporal-booking replicas 0 -n temporal-booking
```

```bash
kubectl scale deployment temporal-booking replicas 3 -n temporal-booking
```

## Remove Cloud CA Certificate
Once you have confirmed workers are all using the new certificate, we can safely delete the old cloud CA certificate.

```bash
$ tcld n ca remove --namespace temporal-booking.sdvdw -f /path/to/OLD-Booking.crt
```

## Sample Kubernetes Worker Deployment
Example of a sample Kubernetes worker deployment, which injects certificate from secret.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/build: "1"
    app.kubernetes.io/component: worker
    app.kubernetes.io/name: temporal-booking-worker
    app.kubernetes.io/version: v2.0
  name: temporal-booking-worker
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: worker
      app.kubernetes.io/name: temporal-booking-worker
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
        app.kubernetes.io/name: temporal-booking-worker
        app.kubernetes.io/version: v2.0
    spec:
      containers:
      - env:
        - name: CHATGPT_API_KEY
          valueFrom:
            secretKeyRef:
              key: KEY
              name: chatgpt-key
        - name: TEMPORAL_HOST_URL
          value: temporal-booking.sdvdw.tmprl.cloud:7233
        - name: TEMPORAL_TASK_QUEUE
          value: temporal-booking
        - name: TEMPORAL_NAMESPACE
          value: temporal-booking.sdvdw
        - name: TEMPORAL_MTLS_TLS_CERT
          value: /etc/certs/tls.crt
        - name: TEMPORAL_MTLS_TLS_KEY
          value: /etc/certs/tls.key
        image: ktenzer/temporal-booking:v1.0
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
        name: temporal-booking-worker
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
          secretName: temporal-booking-tls
```          

## Summary
In this article we stepped through the process of rotating Temporal Cloud certificates. We saw how to rotate client and Temporal Cloud CA certificates. Finally, we showed how rotate certificates in workers running on Kubernetes.

(c) 2023 Keith Tenzer




