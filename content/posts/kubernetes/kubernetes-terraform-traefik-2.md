---
title: "Using Traefik as Ingress Controller on a Kubernetes cluster with cert-manager | Part 2"
description: "Part 2 of the Kubernetes blog series. In this part we deploy the Traefik dashboard and an example application to our cluster, fully TLS encrypted."
date: 2020-11-22T06:27:18+01:00
draft: false
toc: false
images:
  - /images/blog/7/header-image.png
tags:
  - kubernetes
  - terraform
  - server
  - cloud
  - dev-ops
---

## Introduction :pushpin:

This is Part 2 of the Kubernetes blog series. In this part, we will deploy and expose the Traefik dashboard to a custom domain. As a bonus, we will deploy a small example application as well.

The domains will be pointed to our external load balancer and we will secure them with Let's Encrypt through cert-manager.

To follow along with this part, please [follow Part 1 first](https://rafrasenberg.com/posts/kubernetes-with-terraform-traefik-v2-cert-manager-part-1/).

With that being said, let's deploy some services to our shiny new Kubernetes cluster! :clap:

> Article experience level: **Intermediate**

I categorize every article based on complexity. It's a good way to indicate how well you can follow along with the article since it determines how deep I will explain certain concepts.

**Prerequisites**

- Followed [Part 1](https://rafrasenberg.com/posts/kubernetes-with-terraform-traefik-v2-cert-manager-part-1/)
- Domain name
- kubectl installed
- doctl installed

:point_right: [The Github repo for this blog post](https://github.com/rafrasenberg/kubernetes-terraform-traefik-cert-manager)

## Interacting with our Kubernetes cluster

In order to interact with our cluster from our local machine, we need `kubectl` installed.

The Kubernetes command-line tool, `kubectl`, allows you to run commands against Kubernetes clusters. You can use `kubectl` to deploy applications, inspect and manage cluster resources, and view logs.

Besides the Kubernetes command-line tool, we also need `doctl`. This is the official DigitalOcean command-line client. It uses the DigitalOcean API to provide access to most account and Droplet features.

Please follow the docs to install both of these on your machine:

- [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Install doctl](https://www.digitalocean.com/docs/apis-clis/doctl/how-to/install/)

From this moment on I am assuming that you have successfully installed and configured `kubectl` and `doctl`.

When those are installed please visit the Digital Ocean dashboard and go to your cluster. There you will find a command to connect to your cluster, for easy multiple-cluster management.

![Kubernetes Cluster Digital Ocean Kubectl](/images/blog/7/kubernetes-connect-kubectl.png)

Paste this command in your terminal and connect your cluster.

```sh-session
$ doctl kubernetes cluster kubeconfig save 33d303ec-4b1e-4972-8e9e-670344a64122
```

**Output:**

```sh-session
Notice: Adding cluster credentials to kubeconfig file found in "/home/raf/.kube/config"
Notice: Setting current-context to do-ams3-my-special-cluster
```

Great. Everything is set-up now.

## Testing our cluster and configuration :eyes:

Alright with our Kubernetes cluster successfully connected, let's run some commands to verify if our cluster install went as we expected.

```sh-session
$ kubectl get nodes
```

**Output:**

```sh-session
NAME                            STATUS   ROLES    AGE     VERSION
my-special-cluster-pool-3hkcb   Ready    <none>   3h28m   v1.19.3
my-special-cluster-pool-3hkcw   Ready    <none>   3h29m   v1.19.3
```

As you can see, we can interact with our cluster right now through `kubectl`, so that means we can deploy some services!

In the previous blog post and what you can see back in your Terraform configuration, is that we deployed two services through Helm:

- Traefik
- cert-manager

So let's see if they are up and running in our cluster without any errors. Let's check Traefik first.

We use the `get svc` (shorthand for `get services`) command to list all services in the default namespace. As you might remember from our Terraform configuration, we created a `traefik` namespace and deployed the Traefik Helm repository into that. So therefore we will specify the `-n` namespace flag, to return all services in that namespace.

```sh-session
$ kubectl get svc -n traefik
```

**Output:**

```sh-session
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
traefik   LoadBalancer   10.245.173.208   64.225.81.60   80:31782/TCP,443:31660/TCP   3h31m
```

Awesome! Our Traefik LoadBalancer is successfully running as you can see.

The above output lists the Cluster-IP, the External-IP as well as the target and node-ports the service is running on.

The external IP is the one that is exposed to the outside with the Digital Ocean load balancer behind it. So that will be the IP address where you will be pointing your domain name to.

Now let's quickly go over this again but then for `cert-manager`.

```sh-session
$ kubectl get svc -n cert-manager
```

**Output:**

```sh-session
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
cert-manager           ClusterIP   10.245.118.110   <none>        9402/TCP   3h44m
cert-manager-webhook   ClusterIP   10.245.145.85    <none>        443/TCP    3h44m
```

As you can see, all is up and running! :rocket:

## Deploying the ClusterIssuer

The first thing we need to do first is deploying the ClusterIssuer that will generate free SSL certificates for us.

#### What is a ClusterIssuer?

Issuers, and ClusterIssuers, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests. All `cert-manager` certificates require a referenced issuer that is in a ready condition to attempt to honor the request.

If you want to create a single Issuer that can be consumed in multiple namespaces, you should consider creating a ClusterIssuer resource. This is almost identical to the Issuer resource, however is non-namespaced so it can be used to issue Certificates across all namespaces.

For this, we will be using ClusterIssuer.

Let's create the `post-deployment` folder and in there a `01-cert-manager` folder with a `01-issuer.yml` file:

```sh-session
$ mkdir post-deployment/01-cert-manager && touch post-deployment/01-cert-manager/01-issuer.yml
```

Within the `01-issuer.yml` file, add this configuration:

```yml
---
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: youremail@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: traefik-cert-manager
```

Please make sure you modify the email, this has to be a valid one.

As you can see from the configuration, we are using a `http01` solver here. `cert-manager` offers two methods of validation:

- DNS Validation
- HTTP Validation

When working with different kinds of domains, a HTTP01 validation is the easiest. With a HTTP01 challenge, you prove ownership of a domain by ensuring that a particular file is present at the domain. It is assumed that you control the domain if you are able to publish the given file under a given path.

DNS validation on the other hand is very useful when you are working with an organization's domain name and you would like to deploy services on subdomains like so: `*.yourdomain.com`. You can then generate a wildcard certificate and solve the DNS challenge through services like Cloudflare or AWS Route53.

As you might notice here, you see that the `http01` ingress class is called `traefik-cert-manager`. Where did we see that before?

Ah yes! In the custom values file within our Terraform configuration for the Traefik Helm deployment.

```sh-session
additionalArguments:
  - "--providers.kubernetesIngress.ingressClass=traefik-cert-manager"
```

Now you know what that is for! :smile:

Finally, let's deploy our ClusterIssuer through `kubectl`:

```sh-session
$ kubectl apply -f post-deployment/01-cert-manager/01-issuer.yml
```

Now with our ClusterIssuer successfully in place, we can start generating certificates for our services.

## Exposing the Traefik dashboard

To access the Traefik dashboard, you will need a domain name pointing to the load balancer's external IP. You can check which IP that is with the `kubectl get svc -n traefik` command that we explained earlier.

Then in your registrar panel just add an `A` record pointing to that IP address.

With that done let's create a folder that will hold our certificates and in there the certificate file for the Traefik dashboard:

```sh-session
$ mkdir post-deployment/certificates && touch post-deployment/certificates/traefik-dashboard.yml
```

Now edit the `traefik-dashboard.yml` file and add this:

```yml
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: traefik-dashboard-cert
  namespace: traefik
  labels:
    "use-http01-solver": "true"
spec:
  commonName: traefik.yourdomain.com
  secretName: traefik-dashboard-cert
  dnsNames:
    - traefik.yourdomain.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

Please change the domain/sub-domain name here, to the one you want to use for the Traefik dashboard. If you have done that, run the following command to generate a certificate:

```sh-session
$ kubectl apply -f post-deployment/certificates/traefik-dashboard.yml
```

Now wait around a minute or 2 and then run the following command to check if your certificate is ready and you didn't run into any errors:

```sh-session
$ kubectl -n traefik describe certificate traefik-dashboard-cert
```

**Output:**

```sh-session
...
Status:
  Conditions:
    Last Transition Time:  2020-11-22T10:03:19Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-02-20T09:03:18Z
  Not Before:              2020-11-22T09:03:18Z
  Renewal Time:            2021-01-21T09:03:18Z
  Revision:                1
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    100s  cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  99s   cert-manager  Stored new private key in temporary Secret resource "traefik-dashboard-cert-dcc7c"
  Normal  Requested  99s   cert-manager  Created new CertificateRequest resource "traefik-dashboard-cert-66zld"
  Normal  Issuing    95s   cert-manager  The certificate has been successfully issued
```

Awesome, all done! Let's now create our Traefik deployment files and register the dashboard as an Ingress route so we can reach it through the specified domain name.

We will create a new folder within our `post-deployment` folder and add a `middleware.yml` and `ingress.yml` file:

```sh-session
$ mkdir post-deployment/02-traefik
$ touch post-deployment/02-traefik/01-middleware.yml post-deployment/02-traefik/02-ingress.yml
```

Because the Traefik dashboard is exposed by default, we will add a general Kubernetes secret and a Traefik middleware to create simple basic auth protection.

You can add your own password there of course, by using a tool like `htpassword`. Traefik supports passwords hashed with MD5, SHA1, or BCrypt.

In `middleware.yml` add the following:

```yml
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
data:
  # Login: raf | 123
  users: cmFmOiRhcHIxJGRiN1VMZzFGJDB6OUFtWUVLaWRTQ0h4RkpxODdGYTEKCg==
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: traefik-dashboard-basicauth
  namespace: traefik
spec:
  basicAuth:
    secret: traefik-dashboard-auth
```

Now add the ingress route configuration in `02-ingress.yml`, of course changing your domain name again:

```yml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: traefik-dashboard-basicauth
          namespace: traefik
      services:
        - name: api@internal
          kind: TraefikService
          port: 80
  tls:
    secretName: traefik-dashboard-cert
```

#### Let's deploy :rocket:

Alright with everything prepared now, let's deploy our Ingress route and expose the Traefik dashboard! By running the command below, `kubectl` will first apply the middleware and then deploy the ingress route.

```sh-session
$ kubectl apply -f post-deployment/02-traefik/
```

BOOM! We got it up and running. :dart:

If you follow the domain name you will be greeted with a TLS encrypted route to the Traefik dashboard. After entering the basic auth username and password, the dashboard will be displayed. Awesome!

![Traefik Dashboard](/images/blog/7/traefik-dashboard-1.png)

As you can see when you view the HTTP routers, you can see our dashboard route is safely secured with TLS and Traefik recognizes it as a Kubernetes route.

![Traefik HTTP Routers](/images/blog/7/traefik-dashboard-2.png)

## An example application

With the dashboard up and running, let's deploy an example `whoami` application with TLS encryption. The first thing you have to do again, is to add a DNS entry for the `whoami` service. Then add the appropriate folder and files:

```sh-session
$ mkdir post-deployment/03-whoami
$ touch post-deployment/certificates/whoami.yml
$ touch post-deployment/03-whoami/01-whoami.yml post-deployment/03-whoami/02-ingress.yml
```

Time to create a new certificate. So add this to `post-deployment/certificates/whoami.yml`:

```yml
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: whoami-cert
  namespace: whoami
  labels:
    "use-http01-solver": "true"
spec:
  commonName: whoami.rafrasenberg.com
  secretName: whoami-cert
  dnsNames:
    - whoami.rafrasenberg.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

Then add `01-whoami.yml` inside the `post-deployment/03-whoami` folder:

```yml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  namespace: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: containous/whoami
          imagePullPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami
  labels:
    app: whoami
spec:
  type: ClusterIP
  ports:
    - port: 80
      name: whoami
  selector:
    app: whoami
```

Then add the ingress route in `02-ingress.yml`:

```yml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
  namespace: whoami
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`whoami.rafrasenberg.com`)
      kind: Rule
      services:
        - name: whoami
          port: 80
  tls:
    secretName: whoami-cert
```

Alright so before we are generating the certificate again, don't forget to create the `whoami` namespace, since we are deploying to that and it isn't there yet.

```sh-session
$ kubectl create namespace whoami
```

Then generate our certificate again:

```sh-session
$ kubectl apply -f post-deployment/certificates/whoami.yml
```

Wait for some time and see if it is successfully issued:

```sh-session
$ kubectl -n whoami describe certificate whoami-cert
```

If everything thing went well and your certificate was successfully issued, deploy the `whoami` service:

```sh-session
$ kubectl apply -f post-deployment/03-whoami/
```

Great work! :fire: As you can see, the `whoami` service is available through the domain name, running over TLS:

![Whoami Service](/images/blog/7/whoami-service.png)

You can see the service in the Traefik dashboard now as well:

![Whoami Service in Traefik dashboard](/images/blog/7/traefik-dashboard-3.png)

## Conclusion :zap:

Alright so in this blog post we configured our cluster further.

We exposed the Traefik dashboard and deployed an example application. Of course, now it is time to build on this. The issuing of certificates and deployment of the ingress routes, for example, is something that can be automated as well.

However for now we will leave it to this, and hopefully, you learned enough from it so that you can build out your Kubernetes cluster yourself. In the future, I will definitely post some more about it.

See you next time! :wave:

**Sources used for this post**:

- [Kubernetes Docs](https://kubernetes.io/docs/home/)
- [cert-manager Docs](https://cert-manager.io/docs/)
