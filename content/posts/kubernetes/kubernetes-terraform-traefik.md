---
title: "Kubernetes with Terraform & Traefik V2 + cert-manager | Part 1"
description: "Digital Ocean managed Kubernetes through Terraform. Paired with Traefik v2 as the Ingress Controller and cert-manager as the provider for free Let's Encrypts certificates!"
date: 2020-11-21T06:28:07+01:00
draft: false
toc: false
images:
  - /images/blog/6/header-image.png
tags:
  - kubernetes
  - terraform
  - server
  - cloud
  - dev-ops
---

## Introduction :pushpin:

Today we are setting up a managed Kubernetes cluster & load balancer on DigitalOcean using Terraform. Our cluster will be powered with Traefik v2 as our Ingress controller **and** cert-manager to provide us with free Let's Encrypt certificates.

Woaw. That's a mouth full. Well don't worry, it sounds more difficult than it actually is!

In this part, we'll go over the Kubernetes set-up first, and then in Part 2 we will do the deployments. :clap:

> Article experience level: **Intermediate**

I categorize every article based on complexity. It's a good way to indicate how well you can follow along with the article since it determines how deep I will explain certain concepts.

**Prerequisites**

- Terraform > 0.13 installed
- Digital Ocean account

:point_right: [The Github repo for this blog post](https://github.com/rafrasenberg/kubernetes-terraform-traefik-cert-manager)

## Infrastructure as Code (IaC) :construction:

Back in the day, managing IT infrastructure was a tough job. Sysadmins had to manually configure and manage all the hardware and software that was needed for applications to run.

In recent years, this has changed. Cloud computing has vastly improved and it changed the way organizations design, develop, and maintain their IT infrastructure.

#### What is IaC?

_IaC according to Wikipedia:_

Infrastructure as code is the process of managing and provisioning computer data centers through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.

A little complex and full of buzzwords, so let's rewrite that into something short and understandable:

> Infrastructure as code (IaC) means to manage your IT infrastructure using configuration files.

Before IaC, IT personnel would have to manually change configurations to manage their infrastructure. With IaC, your infrastructure’s configuration takes the form of a code file.

Since it’s just text, it’s easy for you to edit, copy, and distribute it. You can **and should** put it under source control, like any other source code file.

![Infastructure as Code](/images/blog/6/infrastructure-as-code.png)

[Image source](https://www.softobiz.com/infrastructure-as-code-an-essential-devops-practice/)

The benefits of Infrastructure as Code are:

- Faster development
- More consistency
- Accountability
- Higher efficiency
- Lower costs

In this blog post, we will use the famous IaC tool, Terraform. So let's start! :rocket:

## Choosing the cloud provider :fish:

For this blog post, we are picking a managed Kubernetes cluster on Digital Ocean, since it is perfect to just start out and explore Kubernetes.

It's a cheaper alternative than AWS EKS since launching a managed cluster + load balancer will only cost you around ~$30 per month whereas AWS EKS will set you back at least $75 per month ($0.10 / hour). And this excludes the Load Balancer, NAT gateway etc.

So for just hacking around, I prefer Digital Ocean as it is an easy and inexpensive way to get started. But for critical production environments, I would not advise using Digital Ocean because compared to AWS EKS it doesn't really compete when looking at flexibility, scalability, and maturity.

You don't want to pay? I get it! If you sign up [through this link](https://m.do.co/c/52031fcadf3c), you will get $100 worth of credit for free. Which means 3 full months of free Kubernetes! :heart:

## IaC with Terraform

From this point on I am going to assume you have Terraform > 0.13 installed and you have set-up your Digital Ocean account.

#### What is Terraform?

Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing, popular service providers as well as custom solutions. :white_check_mark:

The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.

## Getting started with Terraform

Alright, so the first thing we are going to do is prepare our workspace. Let's create a folder and `cd` into it.

```sh-session
$ mkdir do-k8-config && cd do-k8-config
```

The first file we are going to create is `01-backend.tf`. This will basically hold the main configuration files and will tell us which version of Terraform to use and which providers to use.

A provider is responsible for understanding API interactions and exposing resources. Most providers configure a specific infrastructure platform (either cloud or self-hosted).

```hcl
terraform {
  required_version = "~> 0.13.5"
  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0.2"
    }
  }
}
```

Next up is creating the `terraform.tfvars` file. This will hold all the variables of the infrastructure. Terraform automatically loads the variables in this file when running the `terraform apply` command, without needing the specify the `-var-file="foo.tfvars"` flag.

Best practices when working with larger codebases would be to split this variable file into several variable files each corresponding to a certain part of your configuration. But for the simplicity of this blog post, we will store all of it in one file.

The first variable we need to specify is the API token of Digital Ocean. This is needed so Terraform can communicate through the Digital Ocean provider.

To get the Digital Ocean API token, log in, and find your account settings. Generate one if you hadn't already. Then replace the variable below with your token.

```hcl
# 1 Backend variables
do_token                              = "478a6caadc70293857235kshdglkjshasgklasg7cb643525asdgcd1"

```

Now we create the `provider.tf` file.

Terraform configurations must declare which providers they require so that Terraform can install and use them. Additionally, some providers require configuration (like endpoint URLs or cloud regions) before they can be used.

```hcl
# Register the DO token
variable "do_token" {
  type        = string
  description = "Digital Ocean token."
}

# Configure the DigitalOcean Provider
provider "digitalocean" {
  token = var.do_token
}
```

Your current folder structure should look like this right now:

```sh-session
.
└── do-k8-config/
    ├── 01-backend.tf
    ├── provider.tf
    └── terraform.tfvars
```

## Configuring the cluster

Next up is our cluster configuration. Create a file and name it `02-cluster.tf`. In there we first declare the variables that we are going to use. Some variables that we are declaring here are the cluster name, the region and the node size.

```hcl {linenos=table,linenostart=1}
# Variable declaration
variable "cluster_name" {
  type        = string
  description = "Cluster name that will be created."
}
variable "cluster_region" {
  type        = string
  description = "Cluster region."
}
variable "cluster_tags" {
  type        = list(string)
  description = "Cluster tags."
}
variable "node_size" {
  type        = string
  description = "The size of the nodes in the cluster."
}
variable "node_max_count" {
  type        = number
  description = "Maximum amount of nodes in the cluster."
}
variable "node_min_count" {
  type        = number
  description = "Minimum amount of nodes in the cluster."
}
```

Next up is the actual cluster configuration and the connection to the Kubernetes and Helm (the package manager) providers.

If you really want to dive more into Terraform configuration, I highly suggest to check out the docs of each provider, since it is too much to go over all the config settings here in this blog. The docs are really good!

- [Digital Ocean Provider Docs](https://registry.terraform.io/providers/digitalocean/digitalocean/latest/docs)
- [Helm Provider Docs](https://registry.terraform.io/providers/hashicorp/helm/latest/docs)
- [Kubernetes Provider Docs](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)

```hcl {linenos=table,linenostart=27}
# Enable auto upgrade patch versions
data "digitalocean_kubernetes_versions" "do_cluster_version" {
  version_prefix = "1.19."
}

# Create the cluster with autoscaling on
resource "digitalocean_kubernetes_cluster" "do_cluster" {
  name         = var.cluster_name
  region       = var.cluster_region
  auto_upgrade = true
  version      = data.digitalocean_kubernetes_versions.do_cluster_version.latest_version
  tags         = var.cluster_tags

  node_pool {
    name       = "${var.cluster_name}-pool"
    size       = var.node_size
    min_nodes  = var.node_min_count
    max_nodes  = var.node_max_count
    auto_scale = true
  }
}

# Load and connect to Kubernetes
provider "kubernetes" {
  version          = "~> 1.13.3"
  load_config_file = false
  host             = digitalocean_kubernetes_cluster.do_cluster.endpoint
  token            = digitalocean_kubernetes_cluster.do_cluster.kube_config[0].token
  cluster_ca_certificate = base64decode(
    digitalocean_kubernetes_cluster.do_cluster.kube_config[0].cluster_ca_certificate
  )
}

# Load and connect to Helm
provider "helm" {
  kubernetes {
    load_config_file = false
    host             = digitalocean_kubernetes_cluster.do_cluster.endpoint
    token            = digitalocean_kubernetes_cluster.do_cluster.kube_config[0].token
    cluster_ca_certificate = base64decode(
      digitalocean_kubernetes_cluster.do_cluster.kube_config[0].cluster_ca_certificate
    )
  }
  version = "~> 1.3.2"
}
```

Next up is updating our `terraform.tfvars` file to include the new variables. Specify here things like the region where you want to launch your cluster and the node size. We will be using Digital Ocean's autoscaling feature, so therefore we are specifying the max and min node count as well.

```hcl
# 1 Backend variables
do_token                              = "478a6caadc70293857235kshdglkjshasgklasg7cb643525asdgcd1"

# 2 Cluster variables
cluster_name                          = "my-special-cluster-name"
cluster_region                        = "ams3"
cluster_tags                          = ["foo", "development"]
node_size                             = "s-1vcpu-2gb"
node_min_count                        = 2
node_max_count                        = 4
```

## Read, set.. LAUNCH! :rocket:

Before we can deploy our cluster, we will need to run the `terraform init` command.

This will initialize our Terraform workspace. It is used to initialize a working directory containing Terraform configuration files. This is the first command that should be run after writing a new Terraform configuration or cloning an existing one from version control.

```sh-session
$ terraform init
```

As you can see now, a new folder called `.terraform` is formed, holding the configuration files. Next up is planning our execution plan with the `terraform plan` command.

The `terraform plan` command is used to create an execution plan. Terraform performs a refresh, unless explicitly disabled, and then determines what actions are necessary to achieve the desired state specified in the configuration files.

This command is a convenient way to check whether the execution plan for a set of changes matches your expectations without making any changes to real resources or to the state.

The optional `-out` argument can be used to save the generated plan to a file for later execution with terraform apply, which can be useful when running Terraform in automation.

If Terraform detects no changes to the resource or the root module output values, the Terraform plan will indicate that no changes are required.

So let's create our execution plan! :zap:

```sh-session
$ terraform plan -out=terraform.tfplan
```

If this lead to no errors then everything went well! :smile: You can always check your terminal output so see which changes Terraform is going to apply.

We can now officially deploy our cluster by applying our `terraform.tfplan` with the following command:

```sh-session
$ terraform apply "terraform.tfplan"
```

This will take some minutes to finish. When logging in to Digital Ocean, you can see your cluster will now be up and running. AWESOME!

![Digital Ocean Kubernetes Cluster](/images/blog/6/digital-ocean-kubernetes-cluster.png)

**NOTE:**

The state is now stored locally in `terraform.tfstate`. When working with Terraform in a team, use of a local file makes Terraform usage complicated because each user must make sure they always have the latest state data before running Terraform and make sure that nobody else runs Terraform at the same time.

In that case, it is better to store this state remote by using remote state. Terraform writes the state data to a remote data store, which can then be shared between all members of a team. So for best practices when working with production code and a team, use remote state. To keep this tutorial a little shorter I am using the default local `terraform.tfstate`

## Adding Traefik v2 as Ingress controller

Alright with our Kubernetes cluster launched right now, let's add Traefik as our Ingress controller.

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

We will be installing Traefik v2 through the official Helm repository. Helm is a Kubernetes package manager and helps you easily define, install, and upgrade even the most complex Kubernetes applications.

To make Digital Ocean Kubernetes work with the Traefik Helm repository, we need some custom configuration. Create a folder called `helm-values` and within that folder create a file called `traefik.yml`.

```sh-session
$ mkdir helm-values && cd helm-values && touch traefik.yml
```

In this `traefik.yml` file add the following configuration below. This will make sure everything will work properly with `cert-manager`, which we will configure later on. It also enables the dashboard and will automatically redirect all traffic to TLS.

```yml
ingressRoute:
  dashboard:
    enabled: true
    annotations: { traefik.ingress.kubernetes.io/router.tls: "true" }

ports:
  web:
    redirectTo: websecure

additionalArguments:
  - "--log.level=INFO"
  - "--entrypoints.websecure.http.tls"
  - "--providers.kubernetesIngress.ingressClass=traefik-cert-manager"
  - "--ping"
  - "--metrics.prometheus"
```

Now it is time to configure our Terraform files. Create a file called `03-ingress.tf`. Like in our previous Terraform files, first declare the variables that we will need:

```hcl {linenos=table,linenostart=1}
# Variable declaration
variable "ingress_gateway_chart_name" {
  type        = string
  description = "Ingress Gateway Helm chart name."
}
variable "ingress_gateway_chart_repo" {
  type        = string
  description = "Ingress Gateway Helm repository name."
}
variable "ingress_gateway_chart_version" {
  type        = string
  description = "Ingress Gateway Helm repository version."
}
```

In the following section, we will first create a Kubernetes namespace for our Traefik service and then deploy it through Helm. As you can see in here we also specify the custom `traefik.yml` config that we created earlier.

```hcl {linenos=table,linenostart=15}
# Create Traefik namespace
resource "kubernetes_namespace" "ingress_gateway_namespace" {
  metadata {
    annotations = {
      name = "traefik"
    }
    name = "traefik"
  }
}

# Deploy Ingress Controller Traefik
resource "helm_release" "ingress_gateway" {
  name      = var.ingress_gateway_chart_name
  chart     = var.ingress_gateway_chart_repo
  namespace = "traefik"

  values = [
    file("helm-values/traefik.yml")
  ]
}
```

At last, we need to update our `terraform.tfvars` again to enter the new variables.

```hcl
# 1 Backend variables
do_token                              = "478a6caadc70293857235kshdglkjshasgklasg7cb643525asdgcd1"

# 2 Cluster variables
cluster_name                          = "my-special-cluster-name"
cluster_region                        = "ams3"
cluster_tags                          = ["foo", "development"]
node_size                             = "s-1vcpu-2gb"
node_min_count                        = 2
node_max_count                        = 4

# 3 Ingress variables
ingress_gateway_chart_name            = "traefik"
ingress_gateway_chart_repo            = "https://helm.traefik.io/traefik"
ingress_gateway_chart_version         = "9.8.3"
```

Now it's time to create our execution plan and deploy our new Ingress controller to our Kubernetes cluster! As a sanity check, this is what your current folder structure should look like:

```sh-session
.
└── do-k8-config/
    ├── .terraform/
    ├── helm-values/
    │   └── traefik.yml
    ├── 01-backend.tf
    ├── 02-cluster.tf
    ├── 03-ingress.tf
    ├── provider.tf
    ├── terraform.tfstate
    ├── terraform.tfplan
    └── terraform.tfvars
```

Alright with that out of the way, let's create our new plan:

```sh-session
$ terraform plan -out=terraform.tfplan
```

If everything went well, it's time to deploy it!

```sh-session
$ terraform apply "terraform.tfplan"
```

Hoorah, Traefik is deployed!! :rocket:

If you check your Digital Ocean dashboard right now and go to the Networking menu, and then the Load Balancers. You'll see a shiny fresh new load balancer there. I hear you thinking, huh? But we didn't deploy that?

![Digital Ocean Load Balancer](/images/blog/6/digital-ocean-load-balancer.png)

This is the result of using the managed service by Digital Ocean. Whenever a Kubernetes service is declared as type LoadBalancer:

```yml
kind: Service
apiVersion: v1
spec:
  type: LoadBalancer
```

Then Digital Ocean automatically triggers the creation of that load balancer when you deploy the service. To read more about load balancing on Digital Ocean Kubernetes, [follow this link.](https://www.digitalocean.com/docs/kubernetes/how-to/configure-load-balancers/)

## Installing cert-manager

The last Terraform configuration that we have to do is that of `cert-manager`.

Cert-manager builds on top of Kubernetes, introducing certificate authorities and certificates as first-class resource types in the Kubernetes API. This makes it possible to provide 'certificates as a service' to developers working within your Kubernetes cluster.

Let's create a new file called `04-cert-manager.tf`. Like we did before, first declare the variables:

```hcl {linenos=table,linenostart=1}
# Variable declaration
variable "cert_manager_chart_name" {
  type        = string
  description = "Cert Manager Helm name."
}
variable "cert_manager_chart_repo" {
  type        = string
  description = "Cert Manager Helm repository name."
}
variable "cert_manager_chart_version" {
  type        = string
  description = "Cert Manager Helm version."
}
```

Then we create a new namespace for `cert-manager` within Kubernetes and we deploy it through Helm.

```hcl {linenos=table,linenostart=15}
# Create cert manager namespace
resource "kubernetes_namespace" "cert_manager_namespace" {
  metadata {
    annotations = {
      name = "cert-manager"
    }
    name = "cert-manager"
  }
}

# Install helm release Cert Manager
resource "helm_release" "cert-manager" {
  name       = var.cert_manager_chart_name
  chart      = var.cert_manager_chart_name
  repository = var.cert_manager_chart_repo
  version    = var.cert_manager_chart_version
  namespace  = "cert-manager"

  set {
    name  = "installCRDs"
    value = "true"
  }
}
```

Then we need to update our `terraform.tfvars` again to enter the new variables.

```hcl
# 1 Backend variables
do_token                              = "478a6caadc70293857235kshdglkjshasgklasg7cb643525asdgcd1"

# 2 Cluster variables
cluster_name                          = "my-special-cluster-name"
cluster_region                        = "ams3"
cluster_tags                          = ["foo", "development"]
node_size                             = "s-1vcpu-2gb"
node_min_count                        = 2
node_max_count                        = 4

# 3 Ingress variables
ingress_gateway_chart_name            = "traefik"
ingress_gateway_chart_repo            = "https://helm.traefik.io/traefik"
ingress_gateway_chart_version         = "9.8.3"

# 4 Cert manager variables
cert_manager_chart_name               = "cert-manager"
cert_manager_chart_repo               = "https://charts.jetstack.io"
cert_manager_chart_version            = "1.0.4"
```

And at last, you guessed it right, create our execution plan again!

```sh-session
$ terraform plan -out=terraform.tfplan
```

Now let's deploy cert-manager.

```sh-session
$ terraform apply "terraform.tfplan"
```

Your Kubernetes cluster is now up and running with Traefik v2 as the Ingress Controller and cert-manager installed, ready to generate free Let's Encrypt certificates for the domains you will later point to your cluster.

## Conclusion :zap:

Alright so we have our Kubernetes cluster up and running and Traefik and cert-manager are installed. Now in Part 2 of this blog, we will configure cert-manager so we will be able to issue free Let's Encrypt certificates.

We will then expose the Traefik dashboard to the internet and deploy a simple example app as well, both running behind the Digital Ocean load balancer and fully TLS encrypted. Exciting!

For Part 2, [follow this link.](https://rafrasenberg.com/posts/using-traefik-as-ingress-controller-on-a-kubernetes-cluster-with-cert-manager-part-2/)

**Sources used for this post**:

- [Terraform Docs](https://www.terraform.io/docs/index.html)
- [Stackify Article](https://stackify.com/what-is-infrastructure-as-code-how-it-works-best-practices-tutorials/)
