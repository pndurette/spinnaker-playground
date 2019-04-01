# spinnaker-playground

Hello! Here you'll find a way to quickly (~30min) setup a local [Spinnaker](https://www.spinnaker.io) instance (with examples) to play with. This repo was made as a takeaway from my [DevOps Toronto Meetup](https://www.meetup.com/DevOpsTO/) talk '[An Overview of Spinnaker](http://decks.pierre-nick.com/201904_Spinnaker_DevOpsTO/)' of April 2019.

**TL;DR:** Run every code block in a terminal (with Homebrew installed) and go to http://localhost:9000



### In a nutshell

This will guide you to:

* Setup a local lightweight Kubernetes, using:
  * Canonical [`multipass`](https://github.com/CanonicalLtd/multipass) (Ubuntu virtual machine manager)
  * Rancher [k3s](https://github.com/rancher/k3s) (lighweight Kubernetes)
* Installing Spinnaker (via the [stable/spinnaker](https://github.com/helm/charts/tree/master/stable/spinnaker) Helm chart)

And optiinally:

* Installing Spinnaker Tools:
  * [Halyard](https://www.spinnaker.io/reference/halyard/) (`hal`), the Spinnaker CLI config tool
  * [`spin`](https://www.spinnaker.io/guides/spin/), the Spinnaker CLI app/pipeline mangement tool
* Setup an example Application and Pipelines
* Cleaning up



---



## Setup local Kubernetes

### Install `multipass`

```bash
brew cask install multipass
```

If you're not using Homebrew, see [`multipass` readme](https://github.com/CanonicalLtd/multipass) for alternative installation methods

### Create Virtual Machine

```bash
# Spinnaker spins up a few microservices so good cpu/memory/disk helps.
multipass launch --name k3s-spin --cpus 4 --mem 8G --disk 20G
```

**Hint:** Monitor processes and resources of your VM with `htop`: `multipass exec k3s-spin -- sh -c "sudo snap install htop"` then (in a new shell) `multipass exec k3s-spin -- sh -c "htop"`

### Install k3s

```
multipass exec k3s-spin -- sh -c "curl -sfL https://get.k3s.io/ | sh -"
```

### Install and configure `kubectl`

```
brew install jq kubernetes-cli
```

If you're not using Homebrew, see [`jq`](https://stedolan.github.io/jq/download/), [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for alternative installation methods

```bash
# Copy kubeconfig from VM
multipass copy-files k3s-spin:/etc/rancher/k3s/k3s.yaml $HOME/.kube/k3s-spin.yaml

# Get VM IP
K3S_IP=$(multipass info k3s-spin --format json | jq -r '.info."k3s-spin".ipv4[0]')

# Replace 'localhost' in kubeconfig with VM IP
sed -i "s/localhost/${K3S_IP}/g" $HOME/.kube/k3s-spin.yaml

# Tell kubectl to use the k3s kubeconfig by default
export KUBECONFIG=$HOME/.kube/k3s-spin.yaml
```

One last step, tell Kubernetes to use local storage (disk) — You don't need to know what any of this does for this playground, but see [Rancher's Local Path Provisioner](https://github.com/rancher/local-path-provisioner) if you're curious.

```bash
# Get Local Path Provisioner (local-path)
curl -LO  https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Install 
kubectl apply -f local-path-storage.yaml

# Make local-path default storage class
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Cleanup 
rm local-path-storage.yaml
```

**Hint:** Test your Kubernetes connection with `kubectl get all --all-namespaces` (you should get a list of Kubernetes resources). 

---



## Installing Spinnaker

```bash
# The Kubernetes namespace we'll use throughout
export SPIN_NAMESPACE='spin-playground'

# Create namespace
kubectl create ns ${SPIN_NAMESPACE}
```

**NB:** Even though it exits right away, <u>THE FOLLOWING STEP TAKES A WHILE!</u> 

```bash
# Install Spinnaker via k3s' HelmChart resource
cat <<EOF | kubectl create -f -
apiVersion: k3s.cattle.io/v1
kind: HelmChart
metadata:
  name: spinnaker
  namespace: kube-system
spec:
  chart: stable/spinnaker
  targetNamespace: ${SPIN_NAMESPACE}
EOF
```

**Hint:** Monitor the progression of the installation:

* Using `htop` installed earlier (memory usage and cpu usage will creep up), and;
* Using `kubectl`:
  * `kubectl get all --namespace $SPIN_NAMESPACE` (until all deployments are ready);
  * `kubectl logs <resource> -f --namespace $SPIN_NAMESPACE` (to check logs) or;
  * `kubectl describe <resource> --namespace $SPIN_NAMESPACE` (to check states)

**Hint:** The [helm chart](https://github.com/helm/charts/tree/master/stable/spinnaker) will create, in order:

* A Job (`job.batch/spinnaker-install-using-ha`) to bootstrap the rest of the installation;
* Services: `service/spinnaker-minio`, `service/spinnaker-redis-master`, `service/spinnaker-spinnaker-halyard`, respectively: an AWS S3-compatible block storage (for storing config), a cache server (for caching infrastructure), and the Spinaker configuration tool;
* After some time: all of the Spinnaker microservices components as Services, i.e. [Clouddriver](https://github.com/spinnaker/clouddriver), [Orca](https://github.com/spinnaker/orca), [Deck](https://github.com/spinnaker/deck), [Igor](https://github.com/spinnaker/igor), [Gate](https://github.com/spinnaker/gate), [Echo](https://github.com/spinnaker/echo), [Font50](https://github.com/spinnaker/front50), [Rosco](https://github.com/spinnaker/rosco)..  These take some time to all report as Ready too as many depend on others.

**<u>Caveat / Help / My Spinnaker install dies</u>:**

This is a very low-resource install of Spinnaker. Running it is fine, but installing it first does take a toll that can result in some timeouts.

* **Sometimes Minio and Redis Services will die** and go away, if this happens, delete the helm chart resource and add it again:

  `kubectl delete helmcharts/spinnaker --namespace kube-system`

  Then re-run the last step above.

* **Some Spinnaker microservices might `CrashLoopBackOff`** (same reason as above): It should fix itself with time. There's a lot going on and most component depends on others, but the Kubernetes scheduler decides the order with which to create them.

* Upon inspecting their logs, **some Spinnaker microservices might be stuck in error loops** (especially after most other components are up): kill the pod (after looking it up using `kubectl get all --namespace $SPIN_NAMESPACE` . The deployment will replace it.
  e.g.: ` kubectl delete pod/spin-gate-6778864f66-qjqlt --namespace $SPIN_NAMESPACE`



### Connect to your Spinnaker

**NB:** Only proceed once `kubectl get all --namespace $SPIN_NAMESPACE` show all deployments in Ready state!

#### Spinnaker UI

In a new terminal window, port-forward the Spinnaker UI (Deck, port 9000).

```bash
export KUBECONFIG=$HOME/.kube/k3s-spin.yaml
export SPIN_NAMESPACE='spin-playground'

export DECK_POD=$(kubectl get pods --namespace ${SPIN_NAMESPACE} -l "cluster=spin-deck" -o jsonpath="{.items[0].metadata.name}")

kubectl port-forward --namespace ${SPIN_NAMESPACE} $DECK_POD 9000
```

Head to http://localhost:9000. It's up! 🎉

#### Spinnaker API

In a new terminal window, port-forward the Spinnaker API (Gate, port 8084).

```bash
export KUBECONFIG=$HOME/.kube/k3s-spin.yaml
export SPIN_NAMESPACE='spin-playground'

export GATE_POD=$(kubectl get pods --namespace ${SPIN_NAMESPACE} -l "cluster=spin-gate" -o jsonpath="{.items[0].metadata.name}")

kubectl port-forward --namespace ${SPIN_NAMESPACE} $GATE_POD 8084
```

**Hint:** You only need to do this if you plan to use the `hal` or `spin` CLIs.

---

## Install Spinnaker Tools

### Halyard (`hal`) CLI

Follow the [official instructions](https://www.spinnaker.io/setup/install/halyard/#install-halyard-on-docker) (Docker method recommended).

`hal` is the CLI tool used to configure and managing Spinnaker, for example:

* Updating Spinnaker;
* Connecting to a CI engine like Jenkins or Travis;
* Adding a private Docker registry;
* Connecting Spinnaker to a cloud provider such as AWS;
* Backing up and restoring configuration;
* ... and much more: [command reference](https://www.spinnaker.io/reference/halyard/commands).

**NB:** This is optional. The installation used in this playground is self-contained and already adds the current Kubernetes instance as a provider.

**Hint:** For more advanced installation on Kubernetes, use [helm chart values overrides](https://github.com/helm/charts/tree/master/stable/spinnaker). Providers and more can be configured directly on install.

**Hint:** Once you're in the halyard container, any command (e.g. `hal config list`) will let you know your `hal` is working correctly by providing a successful response.

### `spin` CLI

Follow the [official instructions](https://www.spinnaker.io/guides/spin/) (installation and usage). 

**NB:** No need to configure `spin` here. By default, `spin` will look for the Spinnaker API at http://localhost:8084, which is what we have set up.

`spin` is the CLI tool used to manage:

* Applications
* Pipelines
* Pipeline Templates

## Setup an example Application and Pipelines

### Creating the application

```bash
spin application save --file examples/application/my-devopsto-app.json
```

**Hint:** Applications can also be created by:

* Passing attributes directly the attributes to `spin application save`
* In the UI, Applications > Actions > Create Application

## Cleaning up

### Remove Spinnaker from k3s

To remove any trace of Spinnaker from k3s, just delete the namespace it was installed in and the HelmChart resource it was installed with. This can take a while as Kubernetes does a clean removal.

```bash
export SPIN_NAMESPACE='spin-playground'
kubectl delete ns ${SPIN_NAMESPACE}
kubectl delete helmchart spinnaker --namespace kube-system
```

### Remove the entire VM

```bash
multipass delete k3s-spin
multipass purge
```

### Remove `hal`, `spin`



## References

https://medium.com/@zhimin.wen/running-k3s-with-multipass-on-mac-fbd559966f7c

https://medium.com/@zhimin.wen/deploy-jenkins-helm-chart-on-k3s-running-on-macbook-484bb7ba588f



## License

[The MIT License (MIT)](LICENSE) Copyright © 2019 Pierre Nicolas Durette