---
layout: post
title:  "Deploying Kubernetes (Minikube) Locally with OpenTofu"
categories: [OpenTofu, Terraform, Kubernetes]
---

The past 6 months or so, I've been working more closely with Kubernetes in my day job. It's had me interested in using it for my own personal projects. Though I'm going to keep things simple initially with `minikube`, this will be the start of what I hope will be an interesting and productive learning experience.

In this post we'll bring up a local Kubernetes cluster with OpenTofu and Minikube, then bootstrap ArgoCD on it and document some of my preferred initial setup.

### Relevance to Data Engineering and Software Development

I mentioned this briefly in my last post, and I'll say it again: developers who understand infrastructure and how their code is deployed write better software and debug faster than those who don't.

### Tools I will be using in this tutorial and future posts

OpenTofu, Minikube, kubectl, Helm, and ArgoCD are actively used in this post. Istio is enabled and described here because it will show up in later posts.

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

Helm is both a package manager (like `uv`, `conda`, or `poetry`) and a templating software (vaguely similar to jinja) for K8s, making it easier to deploy simple and complex applications alike. It helps you bundle an application image with other dependencies, including sidecars, virtual services, ConfigMaps, and more.

#### ArgoCD

ArgoCD is a K8s-specific tool that constantly checks a git repo for changes, and automatically deploys changes to the K8s cluster based on the changes it sees, thereby keeping git as the source of truth. It has certain advantages over traditional CI/CD pipelines like GitHub Actions, Azure Pipelines, or Jenkins, but is more limited in what it can touch. In general, infrastructure configuration that rarely changes and is global to all applications should be managed by ArgoCD; application-specific deployments should likely be managed via traditional CI/CD.

Be aware that mixing kubectl edits with Helm can be problematic, and performing manual applies when using ArgoCD can cause a constant tension between what exists in git versus what K8s knows the deployed manifest to be.

#### Istio

Istio is a service mesh: a layer that manages how services in the cluster talk to each other and to the outside world. It works by running an Envoy proxy (a "sidecar") next to each application, so features like routing, retries, and encryption don't need to be built into application code. Istio can encrypt traffic between services and verify their identities, restrict which services are allowed to call which others, gradually shift traffic between versions of an app for canary releases, and collect metrics and traces about service-to-service calls, all without changing your application code. It offers far more flexibility than a basic Kubernetes Ingress controller, which only handles traffic entering the cluster, and is widely used in production environments. We enable it now through a Minikube addon so it's ready for later posts, but we won't configure it here.

### Prerequisites

You should be somewhat familiar with Docker and have it installed and running already (`docker info` should succeed). Make sure your Docker settings allow at least 9 GB of memory and 6 CPUs, since the cluster below requests 8192mb and 6 CPUs. If you have less to spare, lower `memory` and `cpus` in `main.tf` below, but note that it may have adverse impacts on future tutorials that refer to this work.

### Setup

For this tutorial, I use [Homebrew](https://brew.sh) as a package manager, but feel free to use your own, and adjust your installation commands accordingly.

1. Install Minikube with `brew install minikube`
1. Pre-download the Kubernetes files the provider needs with `minikube start --vm=true --driver=docker --download-only --kubernetes-version=v1.37.0`. This is required for the tofu/terraform provider to work. We specify `--download-only` so that we don't instantiate a cluster and instead only download the necessary files given the configuration specified (the kubernetes image, drivers, binaries, etc).
1. Install OpenTofu with `brew install opentofu`
1. Install `kubectl` with `brew install kubectl`
1. Install `helm` with `brew install helm`
1. Create a new directory and `cd` into it: `mkdir base-infra && cd $_`
1. Instantiate your git repo: `git init`
1. Create your `.gitignore` file and add the paths listed here to it. State files can contain secrets, which is why they are excluded.
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

If you run into errors, check that the Minikube container is running with `docker ps | grep minikube`.

### Optional: Install kubectx

As you work with more clusters and namespaces, typing `kubectl config use-context ...` and `-n <namespace>` on every command gets tedious. [kubectx](https://github.com/ahmetb/kubectx) is a small tool that makes switching between them quick. Homebrew installs two commands together:

- `kubectx` switches between clusters (kubectl "contexts"), which matters once you have more than one cluster in your kubeconfig.
- `kubens` switches the default namespace, so you don't have to pass `-n` every time.

A cluster is the Kubernetes environment itself; a context is a saved entry in your kubeconfig file (`~/.kube/config`) that pairs a cluster with a user and, optionally, a default namespace. Switching contexts just points kubectl at a different entry in that file.

1. Install it with `brew install kubectx`
1. Try it out:
    ```bash
    kubectx              # list all contexts; the current one is highlighted
    kubectx minikube     # switch to the minikube context
    kubectx -            # switch back to the previous context
    kubens               # list namespaces; the current one is highlighted
    kubens kube-system   # make kube-system the default namespace
    kubens default       # go back to the default namespace
    ```

Once ArgoCD is installed below, you can run `kubens argocd` and drop the `-n argocd` from the commands that follow.

### Install ArgoCD

With the cluster running, we can install ArgoCD onto it. Notice that we're installing it manually with `helm` rather than through OpenTofu. ArgoCD is meant to manage everything deployed to the cluster from git, and that eventually includes itself, so it needs a one-time manual bootstrap. In a future post we'll create an ArgoCD `Application` that points at its own Helm chart, so that ArgoCD manages its own upgrades and configuration from git. If OpenTofu owned the release too, the two tools would end up fighting over it, and you'd have to manually remove it from the tofu/terraform state file anyway.

1. Add the Argo Helm repository and find the current chart version:
    ```bash
    helm repo add argo https://argoproj.github.io/argo-helm # adds the argocd helm chart repo to our host machine's internal index
    helm repo update # pulls in the latest list of charts from the added repos
    helm search repo argo/argo-cd --version 10.10.5
    ```

1. Install the chart into its own namespace:
    ```bash
    helm install argocd argo/argo-cd \
      --namespace argocd \
      --create-namespace \
      --version 10.10.5
    ```

    It may take a moment, but once this completes, helm (hopefully) should have returned the successful deployment status to you, along with instructions for accessing the ArgoCD GUI. We will use the port forward option in the next step.

#### Verify ArgoCD in the browser

ArgoCD's UI isn't exposed outside the cluster, so we use a port-forward to reach it from your machine. If you are unfamiliar with port forwarding, please see my [post on Linux networking basics]({% post_url 2026-04-04-Linux-Networking-Basics %}).

1. Fetch the initial admin password:
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```
1. Forward the ArgoCD server to your machine. Leave this running in its own terminal:
    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
1. Open [https://localhost:8080](https://localhost:8080). Your browser will warn about it being unsecured, which is expected locally. Continue past it and log in with the username `admin` and the password from step 1.

You should see an empty ArgoCD dashboard. We haven't pointed it at a git repo yet, so it has nothing to deploy. We'll do that in a future post, along with handing ArgoCD control of its own chart.

### Tear down

When you're done, you can remove the cluster with `tofu destroy` if desired. Because the cluster is defined in code, you can always bring it back with `tofu apply`. In the future though, remember that destroying the cluster will also destroy any data, logs, and applications stored in it. Manually bootstrapped steps would need to be re-done.

### What's next

We now have a local cluster running with ArgoCD installed, but ArgoCD isn't managing anything yet. In the next post, we'll point it at a git repo so it deploys applications from git, and hand it control of its own Helm chart so that ArgoCD manages itself.

And as always, happy coding!