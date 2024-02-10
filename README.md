## ✨ Features

There are currently 2 different types of configuration available with this template.

1. "**Bare cluster**" - [Talos](https://github.com/siderolabs/talos) that comes with [Cilium](https://github.com/cilium/cilium) installed in the cluster.
2. "**Flux cluster**" - Add-on to the "**Bare cluster**" to deploy an opinionated implementation of [Flux](https://github.com/fluxcd/flux2) using [GitHub](https://github.com/) as GitOps provider and [sops](https://github.com/getsops/sops) to manage secrets.

Each configuration will allow you to activate [Renovate](https://github.com/renovatebot/renovate) which will help manage dependencies via pull requests.

## 💻 Machine Preparation

### Talos

1. Download the latest stable release of Talos from their [GitHub releases](https://github.com/siderolabs/talos/releases). You will want to grab either `metal-amd64.iso` or `metal-rpi_generic-arm64.raw.xz` depending on your system.

2. Take note of the OS drive serial numbers you will need them later on.

3. Flash the iso or raw file to a USB drive and boot to Talos on your nodes with it.

4. Continue on to 🚀 [**Getting Started**](#-getting-started)

## 🚀 Getting Started

Once you have installed Talos on your nodes, there are six stages to getting a Flux-managed cluster up and runnning.

> [!NOTE]
> For all stages below the commands **MUST** be ran on your personal workstation within your repository directory

### 🎉 Stage 1: Create a Git repository

1. Create a new **public** repository by clicking the big green "Use this template" button at the top of this page.

2. Clone **your new repo** to you local workstation and `cd` into it.

3. Continue on to 🌱 [**Stage 2**](#-stage-2-setup-your-local-workstation-environment)

### 🌱 Stage 2: Setup your local workstation

You have two different options for setting up your local workstation.

- First option is using a `devcontainer` which requires you to have Docker and VSCode installed. This method is the fastest to get going because all the required CLI tools are provided for you in my [devcontainer](https://github.com/onedr0p/flux-cluster-template/pkgs/container/flux-cluster-template%2Fdevcontainer) image.
- The second option is setting up the CLI tools directly on your workstation.

#### Devcontainer method

1. Start Docker and open your repository in VSCode. There will be a pop-up asking you to use the `devcontainer`, click the button to start using it.

2. Continue on to 🔧 [**Stage 3**](#-stage-3-bootstrap-configuration)

#### Non-devcontainer method

1. Install the most recent version of [task](https://taskfile.dev/), see the [installation docs](https://taskfile.dev/installation/) for other supported platforms.

    ```sh
    # Homebrew
    brew install go-task
    # or, Arch
    pacman -S --noconfirm go-task && ln -sf /usr/bin/go-task /usr/local/bin/task
    ```

2. Install the most recent version of [direnv](https://direnv.net/), see the [installation docs](https://direnv.net/docs/installation.html) for other supported platforms.

    ```sh
    # Homebrew
    brew install direnv
    # or, Arch
    pacman -S --noconfirm direnv
    ```

    📍 _After `direnv` is installed be sure to **[hook it into your preferred shell](https://direnv.net/docs/hook.html)** and then run `task workstation:direnv`_

3. Install the additional **required** CLI tools

   📍 _**Not using Homebrew or ArchLinux?** Try using the generic Linux task below, if that fails check out the [Brewfile](.taskfiles/Workstation/Brewfile)/[Archfile](.taskfiles/Workstation/Archfile) for what CLI tools needed and install them._

    ```sh
    # Homebrew
    task workstation:brew
    # or, Arch with yay/paru
    task workstation:arch
    # or, Generic Linux (YMMV, this pulls binaires in to ./bin)
    task workstation:generic-linux
    ```

4. Setup a Python virual environment by running the following task command.

    📍 _This commands requires Python 3.11+ to be installed._

    ```sh
    task workstation:venv
    ```

5. Continue on to 🔧 [**Stage 3**](#-stage-3-bootstrap-configuration)

### 🔧 Stage 3: Bootstrap configuration

> [!NOTE]
> The [config.sample.yaml](https://github.com/onedr0p/flux-cluster-template/blob/main/config.sample.yaml) file contain necessary information that is **vital** to the bootstrap process.

1. Generate the `config.yaml` from the [config.sample.yaml](https://github.com/onedr0p/flux-cluster-template/blob/main/config.sample.yaml) configuration file.

    ```sh
    task init
    ```

#### 🔧 Stage 3: Flux

📍 _Using [SOPS](https://github.com/getsops/sops) with [Age](https://github.com/FiloSottile/age) allows us to encrypt secrets and use them with Flux._

1. Create a Age private / public key (this file is gitignored)

    ```sh
    task sops:age-keygen
    ```

2. Fill out the appropriate vars in `config.yaml`

#### Stage 3: Finishing up

1. Complete filling out the rest of the `config.yaml` configuration file.

2. Once done run the following command which will verify and generate all the files needed to continue.

    ```sh
    task configure
    ```

3. Push you changes to git

   📍 **Verify** all the `*.sops.yaml` and `*.sops.yaml` files under `./kubernetes` directory is **encrypted** with SOPS

    ```sh
    git add -A
    git commit -m "Initial commit :rocket:"
    git push
    ```

4.  Continue on to ⚡ [**Stage 4**](#-stage-4-prepare-your-nodes-for-kubernetes)

### ⛵ Stage 4: Install Kubernetes

#### Talos

1. Create Talos Secrets

    ```sh
    task talos:gensecret
    task talos:genconfig
    ```

2. Apply Talos Config

    ```sh
    task talos:apply
    ```

3. Boostrap Talos and get kubeconfig

    ```sh
    task talos:bootstrap
    task talos:kubeconfig node=$master_node_ip_address
    ```

4. Install Cilium and kubelet-csr-approver into the cluster

    ```sh
    task talos:apply-extras
    ```

#### Cluster validation

1. The `kubeconfig` for interacting with your cluster should have been created in the root of your repository.

2. Verify the nodes are online

    📍 _If this command **fails** you likely haven't configured `direnv` as mentioned previously in the guide._

    ```sh
    kubectl get nodes -o wide
    # NAME           STATUS   ROLES                       AGE     VERSION
    # k8s-0          Ready    control-plane,etcd,master   1h      v1.29.1
    # k8s-1          Ready    worker                      1h      v1.29.1
    ```

3. Continue on to 🔹 [**Stage 6**](#-stage-6-install-flux-in-your-cluster)

### 🔹 Stage 6: Install Flux in your cluster

> [!IMPORTANT]
> Skip this stage if you have **disabled** Flux in the `config.yaml`

1. Verify Flux can be installed

    ```sh
    flux check --pre
    # ► checking prerequisites
    # ✔ kubectl 1.27.3 >=1.18.0-0
    # ✔ Kubernetes 1.27.3+k3s1 >=1.16.0-0
    # ✔ prerequisites checks passed
    ```

2. Install Flux and sync the cluster to the Git repository

    ```sh
    task flux:bootstrap
    # namespace/flux-system configured
    # customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
    # ...
    ```

3. Verify Flux components are running in the cluster

    ```sh
    kubectl -n flux-system get pods -o wide
    # NAME                                       READY   STATUS    RESTARTS   AGE
    # helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
    # kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
    # notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
    # source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
    ```

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Flux itself! Merging a PR will cause Flux to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](https://github.com/onedr0p/flux-cluster-template/blob/main/.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state.

1. Start by checking all Flux Kustomizations & Git Repository & OCI Repository and verify they are healthy.

    ```sh
    flux get sources oci -A
    flux get sources git -A
    flux get ks -A
    ```

2. Then check all the Flux Helm Releases and verify they are healthy.

    ```sh
    flux get hr -A
    ```

3. Then check the if the pod is present.

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

4. Then check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    # or
    stern -n <namespace> <fuzzy-name>
    ```

5. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

6. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on NFS. If you are unable to figure out your problem see the help section below.
