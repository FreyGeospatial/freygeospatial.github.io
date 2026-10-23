---
layout: post
title:  "Deploying Kubernetes (Minikube) Locally with OpenTofu"
categories: [OpenTofu, Terraform, Kubernetes]
---

The past 6 months or so, I've been working more closely with Kubernetes in my day job. It's had me interested in using it for my own personal projects. Though I'm going to keep things simple initially with `minikube`, this will be the start of what I hope will be an interesting and productive learning experience.

In this post we'll stand up a local Kubernetes cluster with OpenTofu instead of running `minikube start` by hand. That's more ceremony for a single local cluster, but the configuration lives in git, can be rebuilt or destroyed with one command, and uses the same workflow I'd use for real cloud infrastructure later.

### Relevance to Data Engineering and Software Development

I mentioned this in my last post briefly, and I will again once more: developers who understand infrastructure and how their code is deployed write better software and debug faster than those who don't.

### Tools I will be using in this tutorial and future posts:

Only OpenTofu, Minikube, and kubectl are used in this post. Helm and ArgoCD are installed or described here because they'll show up in later posts.

#### Kubernetes and Minikube

Kubernetes (K8s) is an open source system for automating deployment, scaling, and management of containerized applications (such as those running with Docker). It is a particularly popular tool for managing microservices and is the backend behind many managed cloud services and SaaS platforms.

Minikube is an implementation of K8s that lets you easily deploy a local cluster so that you can focus more heavily on application development. Minikube can run K8s in a VM or in a Docker container. We use the Docker driver, so you must have the Docker daemon running.

While it is possible to [deploy kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way), that will be a future goal.

#### IaC and OpenTofu

Infrastructure as Code (IaC) is the practice of managing and deploying infrastructure via code, as opposed to more manual processes. IaC can be git committed just like application code.

If you have heard of Terraform before, OpenTofu is its open source fork, created after Terraform's license change, and it remains largely compatible with it. It is a declarative IaC tool: you define the desired state of your infrastructure without specifying the steps needed to achieve that state. It handles the underlying API calls to create those resources for you, which is easier to reason about than scripting each step imperatively (for example, with a chain of CLI commands). It is idempotent in its execution and a popular tool among developers. While often used for cloud deployments, it can also be used for on-prem and local deployments.

To keep my work as reproducible as possible and easy to refer back to, I've decided to deploy my Minikube cluster on my local machine with OpenTofu.

#### Kubectl

Kubectl is the main command-line tool used to manage K8s clusters, including Minikube. It interacts with the K8s control plane via the K8s API. Simply, it provides a mechanism for viewing logs, deploying applications, managing networks, and all aspects of the cluster. When Minikube creates a cluster, it adds a context to your kubeconfig so that kubectl talks to it automatically.

#### Helm

Helm is both a package manager (like `uv`, `conda`, or `poetry`) and a templating software (vaguely similar to jinja), making it easier to deploy simple and complex applications alike. It helps you bundle an application image with other dependencies, including sidecars, virtual services, ConfigMaps, and more.

#### ArgoCD

ArgoCD is a K8s specific tool that constantly checks a git repo for changes, and automatically deploys changes to the K8s cluster based on the changes it sees, thereby keeping git as the source of truth. It has certain advantages over traditional CI/CD pipelines like GitHub Actions, Azure Pipelines, or Jenkins, but is more limited in what it can touch. In general, infrastructure configuration that rarely changes and is global to all applications should be managed by ArgoCD; application specific deployments should likely be managed via traditional CI/CD.

Be aware that mixing kubectl edits with Helm can be problematic, and performing manual applies when using ArgoCD can cause a constant tension between what exists in git versus what K8s knows the deployed manifest to be.

### Prerequisites

You should be somewhat familiar with Docker and have it installed and running already (`docker info` should succeed). Make sure your Docker settings allow at least 9 GB of memory and 6 CPUs, since the cluster below requests 8192mb and 6 CPUs. If you have less to spare, lower `memory` and `cpus` in `main.tf` below.

### Setup

For this tutorial, I use [Homebrew](https://brew.sh) as a package manager, but feel free to use your own, and adjust your installation commands accordingly.

1. Install Minikube with `brew install minikube`
1. Pre-download the Kubernetes files the provider needs with `minikube start --vm=true --driver=docker --download-only --kubernetes-version=v1.37.0`. This is required for the tofu/terraform provider to work. We specify `--download-only` so that we don't instantiate a cluster and instead only download the necessary files given the configuration specified (the kubernetes image, drivers, binaries, etc).
1. Install OpenTofu with `brew install opentofu`
1. Install `kubectl` with `brew install kubectl`
1. Install `helm` with `brew install helm`
1. Create a new directory and `cd` into it: `mkdir base-infra && cd $_`
1. Instantiate your git repo: `git init`
1. Create your `.gitignore` file and add the paths listed here to it. State files can contain secrets, which is why they are excluded. Note that `.terraform.lock.hcl` is *not* ignored; commit it so provider versions stay pinned.
    ```txt
    # Local .terraform directories
    .terraform/

    # .tfstate files
    *.tfstate
    *.tfstate.*

    # Crash log files
    crash.log
    crash.*.log

    # Exclude all .tfvars files, which are likely to contain sensitive data, such as
    # password, private keys, and other secrets. These should not be part of version
    # control as they are data points which are potentially sensitive and subject
    # to change depending on the environment.
    *.tfvars
    *.tfvars.json

    # Ignore override files as they are usually used to override resources locally and so
    # are not checked in
    override.tf
    override.tf.json
    *_override.tf
    *_override.tf.json

    # Ignore transient lock info files created by terraform apply
    .terraform.tfstate.lock.info

    # Include override files you do wish to add to version control using negated pattern
    # !example_override.tf

    # Include tfplan files to ignore the plan output of command: terraform plan -out=tfplan
    # example: *tfplan*

    # Ignore CLI configuration files
    .terraformrc
    terraform.rc

    # Optional: ignore graph output files generated by `terraform graph`
    # *.dot

    # Optional: ignore plan files saved before destroying Terraform configuration
    # Uncomment the line below if you want to ignore planout files.
    # planout
    ```
1. Create a folder for your tofu (terraform) code and `cd` into that also: `mkdir tofu && cd $_`
1. Create a `main.tf` and `provider.tf` file and populate them accordingly:

    **main.tf**
    ```hcl
    resource "minikube_cluster" "my_local_cluster" {
      driver       = "docker"
      cluster_name = "minikube"
      cpus         = 6        # Minikube recommends at least 4 cores for istio
      memory       = "8192mb" # Minikube recommends at least 8192mb for istio: https://minikube.sigs.k8s.io/docs/handbook/addons/istio/

      # Enable useful local add-ons automatically, which reduces the need for manual setup.
      # For the full list of add-ons, use `minikube addons list`
      addons = [
        "default-storageclass",
        "dashboard",
        "istio-provisioner",
        "istio",
        "metrics-server",
      ]
    }
    ```
    **provider.tf**
    ```hcl
    terraform {
      required_version = ">= 1.6.0"
      required_providers {
        minikube = {
          source  = "scott-the-programmer/minikube"
          version = "~> 0.8.0"
        }
      }
    }

    provider "minikube" {
      kubernetes_version = "v1.37.0"
    }
    ```
    I've filled in some of the configurations, but don't worry too much about that just yet; we will cover them as needed later. In short: `driver` is how Minikube runs the cluster (Docker, in our case), `cpus` and `memory` are the resources allotted to it, and `addons` are optional cluster components Minikube installs for you.

### Deploy

Run the following from inside the `tofu` directory:

1. `tofu init`: sets up your working directory, downloads the required provider plugins, and initializes the backend state (for our purposes, saved locally).
1. `tofu plan`: shows the changes OpenTofu proposes to make, without making them. You should see one `minikube_cluster` resource to be created.
1. `tofu apply`: shows the plan again and asks for confirmation. Type `yes`. Creating the cluster can take a few minutes.

### Verify

Confirm the cluster is up and kubectl is pointed at it:

```bash
kubectl config current-context   # should print "minikube"
kubectl get nodes                # the node should be Ready
kubectl get pods -A              # system pods should be Running
```

### Tear down

When you're done, remove the cluster with `tofu destroy`. Because the cluster is defined in code, you can always bring it back with `tofu apply`.
