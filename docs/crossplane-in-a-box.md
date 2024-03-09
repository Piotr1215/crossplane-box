---
title: Crossplane in a box
description: Setting up local Crossplane development with KIND cluster, just and ArgoCD
tags: ["kubernetes", "gitops", "crossplane", "argo-cd", "kind", "just"]
hide_table_of_contents: false
---

# Crossplane in a box - setting up local development environment

![crossplane-box](_media/crossplane-box.png)

This article will be helpful for anyone interested in setting up a local Crossplane dev/test environment in a reproducible and easy way.
Source code for this blog is available in a [companion repository](https://github.com/Piotr1215/crossplane-box).

<!--truncate-->

There are several reasons why local Crossplane environment is useful:

- fast prototyping of the infrastructure APIs development
- testing infrastructure changes and configuration
- testing new Crossplane versions
- testing new Crossplane CLI versions

Given that a local or test Crossplane environment is so useful, its creation should be fully automated. Let's learn how to do it.

## Choosing local Kubernetes cluster

First things first, in order to run Crossplane we need a Kubernetes cluster.

There are a lot of choices as it comes to running a local Kubernetes instance, but we are going to focus on KIND.
KIND is very easy to maintain and fast to set up. Second best option is `k3s`.

- [Minikube](https://minikube.sigs.k8s.io/docs/)
- [MicroK8s](https://microk8s.io/)
- **→** [Kind](https://kind.sigs.k8s.io/) Kubernetes IN Docker
- [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Rancher Desktop](https://rancherdesktop.io/)
- [k3s](https://github.com/k3s-io/k3s) also with [k3d](https://k3d.io/v5.4.6/) wrapper

## Imperative vs Declarative

Before we dive into the setup, let's talk about _declarative_ vs _imperative_
approach. Both approaches makes sense depending on the circumstances.
Declarative approach provides more stability and predictability, while
imperative approach is suitable for quick prototyping and provides more control
over the setup.

> Whenever it comes to **something vs something else**, it's really about when it makes more sense to use **something** and when **something else**.

Given this heuristic, we will use both approaches depending on the requirements and what we want to achieve.

## Orchestrating commands

Setting up a local or test Crossplane environment means orchestrating a bunch of commands, including scripts, yaml files, helm charts etc.
In the _imperative_ paradigm, this is typically done via a `Makefile` or `bash scripts`. The problem with `make` is that it is designed as a tool to build C source code, it _can_ run commands but that's not its purpose. This means that when using `Makefile` we take on the whole unnecessary baggage of the build part.

Separate `bash scripts` are a bit better but after a while they became too
verbose and hard to maintain.

There is a tool that combines best of both worlds; [just](https://github.com/casey/just) is similar to `make`, but focused on commands orchestration.

## Declarative approach is still our friend

`Justfile` contains all imperative logic needed to quickly create and destroy
our local cluster. It exposes various knobs for us to interact with it.

Installing a helm chart, operator or simple yaml file can be done declaratively using a GitOps process.
This can be accomplished with `Flux` or `ArgoCD`. For a quick, local setup [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) is a bit more user friendly due to its robust web client.
Here we are utilizing `app of apps` pattern to bootstrap additional apps from a single source. [This article](https://kubito.dev/posts/automated-argocd-app-of-apps-installation/) describes the pattern very well.

> We have just scratched the surface of ArgoCD or GitOps. You can read more about GitOps in [here](https://itnext.io/gitops-with-kubernetes-740f37ea015b) and [here](https://itnext.io/gitopsify-cloud-infrastructure-with-crossplane-and-flux-d605d3043452).

## Setup

I'm using [Universal Crossplane](https://github.com/upbound/universal-crossplane) (uxp) which is an upstream fork of
[Crossplane](https://crossplane.io/), but it would work equally well with Crossplane.

> The setup is tested on macOS and Linux. It should work on Windows with WSL2 as
> well.

### Prerequisites

Other than `kind`, we need a few additional CLI tools:

#### `just` command runner

For macOS users, you can use Homebrew to install `just`:

```bash
brew install just
```

For Linux users, refer to the [just
repository](https://github.com/casey/just#installation) for installation
instructions.

#### `envsubst`

> There are several ways of templating YAML. We can wrap it in a [helm chart](https://helm.sh/docs/topics/charts/), use [ytt](https://carvel.dev/ytt/), [jsonnet](jsonnet), [yq](https://mikefarah.gitbook.io/yq/), [kustomize](https://github.com/kubernetes-sigs/kustomize) or many others. Those are all valid approaches, but for local environment, there is a simpler method.

> We will use `envsubst` instead. It is a part of the [GNU gettext utilities](https://www.gnu.org/software/gettext/manual/gettext.html) and should be already installed on your system.
> This tool enables us to _patch_ environment variables on the fly.

On macOS, you can install `gettext` (which includes `envsubst`) using Homebrew:

    brew install gettext
    brew link --force gettext

On Linux, `envsubst` is usually included with the `gettext` package. Use your
distribution's package manager to install it:

For Debian-based distributions (e.g., Ubuntu):

    sudo apt-get update
    sudo apt-get install gettext

For RHEL-based distributions (e.g., CentOS):

    sudo yum install gettext

#### `kubectl`

Follow the [official Kubernetes
documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to
install `kubectl` for your operating system.

With all the prerequisites installed, we can now proceed to setting up our local Crossplane environment.

### How To

Here are simple steps to get started:

1. Fork the repository `https://github.com/Piotr1215/crossplane-box`
2. Export variable with your GitHub user name `export GITHUB_USER=your-github-username`
3. Run `just setup` to create prerequisites

Running the last command will do the following:

- replace my GitHub user name with yours in the `bootstrap.yaml` and `apps/application_crossplane_resources.yaml`
- create a kind cluster with ArgoCD and UXP installed
- copy ArgoCD server secret to clipboard and launch default browser with ArgoCD login page

> - username: **admin**
> - password: **should be in your clipboard** so just paste it in the `password` text box. In case this didn't work, you can run `@kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d` to get the password.

ℹ️
This setup is meant to be quick and easy, focusing on getting you started without dealing with HTTPS certificates. If you're interested in a more secure setup with HTTPS for local development, you might explore using mkcert. It requires a bit more work, as everyone needs to run mkcert on their machine to create and trust their certificates

Typing `just<TAB>` will show all available just recipes. Here is the list:

```just
Available recipes:
    default        # targets marked with * are main targets
    setup          # * setup kind cluster with uxp, ArgoCD and launch argocd in browser
    setup_kind cluster_name='control-plane' # setup kind cluster
    setup_uxp xp_namespace='crossplane-system' # setup universal crossplane
    setup_argo     # setup ArgoCD and patch server service to nodePort 30950
    launch_argo    # copy ArgoCD server secret to clipboard and launch browser, user admin, pw paste from clipboard
    bootstrap_apps # bootstrap ArgoCD apps and set reconcilation timer to 30 seconds
    teardown       # * delete KIND cluster
```

Now we ready to start iterating quickly on our local Crossplane setup, by adding
new content to the `yaml` directory and pushing changes to the repository. It's
also possible to create a new argocd app in the `app` folder and point it to a
different repository or subfolder.

### Install more content

ArgoCD's `bootstrap` app observes the `apps` directory for any changes once deployed.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps
    repoURL: https://github.com/<your github user/org>/crossplane-box
    targetRevision: HEAD
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

Notice that the `apps` directory already contains an app that points to the
`yaml` folder with crossplane functions and providers. Adding to this directory
and pushing the changes to the repository will automatically deploy the new
content to the cluster.

When you add new content to the `yaml` directory, you can deploy it to the
cluster by running the following command:

```bash
just bootstrap_apps
```

This will deploy all the apps from the `apps` folder and the content of the
`yaml` folder into the cluster via ArgoCD.

Navigate to the ArgoCD web interface and you should see all the resources deployed.

![argo-res](_media/argo-res.png)

### Destroy the cluster

```bash
just teardown
```

## Collaboration

Local setup doesn't exclude collaboration. There are 2 main ways how we can collaborate on demand.

- use ngrok to expose local port on the internet and share our ArgoCD instance and enable someone to see state of our cluster
- accept PRs to our forked repository to let someone else install infra/apps on our cluster or change the setup

> [Read more here](https://itnext.io/expose-local-kubernetes-service-on-internet-using-ngrok-2888a1118b5b) about using ngrok to share a local Kubernetes service over the internet.

## Summary

Deploying a local/test Crossplane instance can and should be fully automated.
We've seen how using _declarative_ and _imperative_ techniques helped us to
create a fully functional cluster with ability to add Crossplane resources and
configuration.

This setup can be easily transitioned to a production cluster. Additionally it's
easy to keep adding new recipes that specialize in different tasks, such as
testing, tracing, debugging, etc.

I'm using this setup to test Universal Crossplane and develop compositions. What will
you use it for?
