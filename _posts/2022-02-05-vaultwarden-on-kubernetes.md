---
layout:     post
title:      Bitwarden on Kubernetes with Vaultwarden
date:       2022-02-04 17:45:00
summary:  Self-hosting Bitwarden on Civo Kubernetes with the alternative implementation of Bitwarden API written in Rust
category: development
tags: [ Kubernetes, Bitwarden, Vaultwarden]
published: true
---

I've migrated to Bitwarden Free version back in March 2021 after Lastpass [changed their policy on their free version](https://support.logmeininc.com/lastpass/help/what-can-i-expect-to-change-for-lastpass-free-on-march-16-2021) which drove most of the users away. The migration was super smooth and Bitwarden made it very easy for the users to migrate into their free service.

Learning Kubernetes at work made me want to take it up a notch by hosting it myself on Kubernetes managed platform as a start. I chose Civo because they offer 250 USD free credit if you sign up and you can have your managed Kubernetes cluster for as little as 5 USD (we'll see!).

Bitwarden offers their own implementation if you want to self-host the server [here](https://github.com/bitwarden/server). I however attracted to this exciting alternative implementation of the Bitwarden API server written in Rust which supposedly made it super light-weight and does not use a lot of resources to run.

The project is called [Vaultwarden](https://github.com/dani-garcia/vaultwarden). It's not an official one but super interesting nonetheless.

## Launching Kubernetes cluster on Civo

So the first step after signing up to Civo is to launch a cluster. It's super straightforward and doesn't require a lot of effort, you can launch your own cluster in less than 10 minutes.

You can read more about it [here](https://www.civo.com/docs/quick-start/600).

Civo has their own marketplace for installing applications when you launch a cluster, so I've picked Traefik for exposing the Vaultwarden [Ingress service](https://kubernetes.io/docs/concepts/services-networking/ingress/) and metrics-server for basic cluster metrics (nodes CPU & memory usage).


## Preparations before installing Vaultwarden

### Create Persistent Volume Claim for SQLite backend for the Bitwarden secrets data

This can be achieved by applying the following manifest:


```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: civo-volume-vaultwarden
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### Add domain and use Civo to manage DNS records

I went ahead and added my domain into Civo and manage the DNS from there, which is one the pre-requisites of using the Okteto's Civo DNS Webhook in later step. I also created CNAME that points to the Kubernetes cluster DNS name (can be retrieved from the Kubernetes Dashboard in Civo)


### Install Cert-Manager

Since I'm using Civo's managed Kubernetes service, this can directly be installed from the [Application Marketplace](https://www.civo.com/learn/deploying-applications-through-the-civo-kubernetes-marketplace) (or can be done when launching the cluster)


### Install Okteto's Civo DNS Webhook

When getting a wildcard certificate, Let's Encrypt asks you to prove that you control the DNS for your domain name by putting a specific value in a TXT record under that domain name. This is known as a DNS01 challenge. cert-manager has support for a few providers out of the box, which you can extend via Webhooks. cert-manager doesn't support Civo out of the box (or at least I wasn't successful with another route I followed), so I went ahead and created one.

To install the webhook, run the commands below which will run the required pods in the `cert-manager` namespace.

```
helm install webhook-civo https://storage.googleapis.com/charts.okteto.com/cert-manager-webhook-civo-0.1.0.tgz --namespace=cert-manager
```

To check all the pods running in the `cert-manager` namespace:

```
❯ kubectl get pods -n cert-manager
NAME                                                      READY   STATUS    RESTARTS   AGE  
cert-manager-5d8b844856-qtnf4                             1/1     Running   0          4d19h
webhook-civo-cert-manager-webhook-civo-5c865bb9b9-dvwrc   1/1     Running   0          4d19h
cert-manager-webhook-8f5767998-qlx8s                      1/1     Running   0          4d19h
cert-manager-cainjector-5fb5c99bf5-vc7ht                  1/1     Running   10         4d19h
```

### Configuring the DNS issuer

Create a secret in your cluster using the command below:

```
kubectl create secret generic civo-dns -n cert-manager --from-literal=key=<YOUR_CIVO_API_KEY>
```

Save the following as `issuer.yaml`

```
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: civo
spec:
  acme:
    email: example@example.com // put in the correct email address here
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - dns01:
        webhook:
          solverName: "civo"
          groupName: civo.webhook.okteto.com
          config:
            apiKeySecretRef:
              key: key
              name: civo-dns
```

Apply it:

```
kubectl apply -f issuer.yaml -n cert-manager
```

Save the following as `certificate.yaml`:

```
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: wildcard-certificate
spec:
  dnsNames:
  - '*.example.com'
  issuerRef:
    kind: Issuer
    name: civo
  secretName: wildcard-example-com-tls
```

Apply it:

```
kubectl apply -f certificate.yaml -n cert-manager
```

To check the status of the requested certificate, we can run the following:

```
kubectl get certificate wildcard-certificate -n cert-manager
```

## Installing Vaultwarden on the Kubernetes cluster with Helm

The easiest way to get Vaultwarden installed on the Kubernetes cluster is with Helm which is Kubernetes package manager and the repository that I'm using in this implementation is the one created by folks at [k8s-at-home project](https://artifacthub.io/packages/helm/k8s-at-home/vaultwarden).

For this to work you need to install Helm locally. I'm using a Mac so it's as simple as running:

```
brew install helm
```

After `helm` is installed, we'll do the following to get Vaultwarden chart:

```
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo update
```

We need to customize this a little bit and make sure we have everything that we need through the `values.yaml`. 

An example of the one I'm using is below. In this implementation, I'm using the default SQLite backend with Kubernetes persistent volume (which we created earlier). Save it as `values.yaml`.

```
image:
  # -- image repository
  repository: vaultwarden/server
  # -- image pull policy
  pullPolicy: IfNotPresent
  # -- image tag
  tag: 1.22.2

strategy:
  type: Recreate

# -- environment variables. See [image docs](https://github.com/dani-garcia/vaultwarden/blob/main/.env.template) for more details.
# @default -- See below
env:
# -- Config dir
  DATA_FOLDER: "config"

# -- Configures service settings for the chart. Normally this does not need to be modified.
# @default -- See values.yaml
service:
  main:
    ports:
      http:
        port: 80
      websocket:
        enabled: true
        port: 3012

ingress:
  # -- Enable and configure ingress settings for the chart under this key.
  # @default -- See values.yaml
  main:
    enabled: false

# -- Configure persistence settings for the chart under this key.
# @default -- See values.yaml
persistence:
  config:
    enabled: true
    type: pvc
    readOnly: false
    storageClass: civo-volume
    existingClaim: civo-volume-vaultwarden
    accessMode: ReadWriteOnce
    size: 5Gi

# https://github.com/bitnami/charts/tree/master/bitnami/mariadb/#installing-the-chart
mariadb:
  enabled: false
  # primary:
  #   persistence:
  #     enabled: true
  # auth:
  #   username: "username"
  #   password: "password"
  #   database: database

# https://github.com/bitnami/charts/tree/master/bitnami/postgresql/#installing-the-chart
postgresql:
  enabled: false
  # postgresqlUsername: ""
  # postgresqlPassword: ""
  # postgresqlDatabase: ""
  # persistence:
  #   enabled: true
  #   storageClass:
  #   accessModes:
  #     - ReadWriteOnce
```

Next we run the following to install Vaultwarden on the Kubernetes cluster:

```
helm install vaultwarden k8s-at-home/vaultwarden -f values.yaml
```

The check the pod and the service that we deployed:

```
❯ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE  
vaultwarden-6886ff6f45-cxqqc   1/1     Running   0          4d19h

❯ kubectl get service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE  
kubernetes    ClusterIP   10.43.0.1       <none>        443/TCP           5d17h
vaultwarden   ClusterIP   10.43.146.202   <none>        80/TCP,3012/TCP   5d15h
```

### Exposing the service through Kubernetes Ingress

We will need to expose the service created above through Ingress so that we can use it through web or clients. So here's the manifest for that:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vaultwarden
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/frontend-entry-points: http,https
    traefik.ingress.kubernetes.io/redirect-entry-point: https
    traefik.ingress.kubernetes.io/redirect-permanent: "true"
spec:
  tls:
    - secretName: wildcard-example-com-tls
      hosts: 
        - bitwarden.example.com
  rules:
  - host: bitwarden.example.com
    http:
      paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vaultwarden
                port:
                  number: 80
          - path: /notifications/hub
            pathType: Prefix
            backend:
              service:
                name: vaultwarden
                port:
                  number: 80
          - path: /notifications/hub/negotiate
            pathType: Prefix
            backend:
              service:
                name: vaultwarden
                port:
                  number: 3012
```

Save it as `ingress.yaml` and apply it with:

```
kubectl apply -f ingress.yaml
```

The CNAME record that was created earlier should now be exposed and we can now access the service at https://bitwarden.example.com.

To check the Ingress from Kubernetes point of view, we can do:

```
kubectl get ingress
```

Later!