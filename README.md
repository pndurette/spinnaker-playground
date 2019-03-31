# spinnaker-playground

Hello! Here you'll find a way to quickly (~30min) setup a local [Spinnaker](https://www.spinnaker.io) instance (with examples) to play with. This repo was made as a takeaway from my [DevOps Toronto Meetup](https://www.meetup.com/DevOpsTO/) talk '[An Overview of Spinnaker](http://decks.pierre-nick.com/201904_Spinnaker_DevOpsTO/)' of April 2019.

**TL;DR:** Run every code block in a terminal (with Homebrew installed) and go to http://localhost:9000



### In a nutshell

This will guide you to:

* Setup a local lightweight Kubernetes, using:
  * Canonical [`multipass`](https://github.com/CanonicalLtd/multipass) (Ubuntu virtual machine manager)
  * Rancher [k3s](https://github.com/rancher/k3s) (lighweight Kubernetes)
* Install Spinnaker (via the [stable/spinnaker](https://github.com/helm/charts/tree/master/stable/spinnaker) Helm chart), and:
  * [Halyard](https://www.spinnaker.io/reference/halyard/) (`hal`), the Spinnaker CLI config tool
  * [`spin`](https://www.spinnaker.io/guides/spin/), the Spinnaker CLI app/pipeline mangement tool
* Set up an Application and Pipelines
* Cleanup



---



## Local Kubernetes

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



## Spinnaker

### Install Spinnaker

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
* After some time: all of the Spinnaker microservices components as Services, i.e. [Clouddriver](https://github.com/spinnaker/clouddriver), [Orca](https://github.com/spinnaker/orca), [Deck](https://github.com/spinnaker/deck), [Igor](https://github.com/spinnaker/igor), [Gate](https://github.com/spinnaker/gate), [Echo](https://github.com/spinnaker/echo), [Font50](https://github.com/spinnaker/front50), [Rosco](https://github.com/spinnaker/rosco)..

**<u>Caveat / Help / My Spinnaker install dies</u>:** Sometimes Minio and Redis will die and go away (possibly because of timeouts), if this happens, delete the helm chart resource and add it again:

`kubectl delete helmcharts/spinnaker --namespace kube-system`

Then re-run the last step above.

---



## Cleanup

### Remove Spinnaker from Kubernetes

To remove any trace from Spinnaker from k3s, just delete the namespace it was installed in and the HelmChart resource it was installed with. This can take a while as Kubernetes does a clean removal.

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



