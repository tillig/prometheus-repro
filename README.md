# prometheus-repro

Reproduction environment for Istio mTLS problem / [Prometheus issue #9068](https://github.com/prometheus/prometheus/issues/9068)

Prerequisite:

- Kubernetes cluster with a reasonably recent version of Kubernetes. I've been using 1.19.11.
- Your `kubectl` context is set to the cluster you're deploying to.
- You have `helm` 3.x installed and on the path. I've been using Helm 3.6.2.

```bash
## Install Istio 1.6.14
# Get istioctl 1.6.14
wget https://github.com/istio/istio/releases/download/1.6.14/istioctl-1.6.14-linux-amd64.tar.gz
tar -xzf istioctl-1.6.14-linux-amd64.tar.gz
chmod +x istioctl

# Precheck, just to make sure
./istioctl exp precheck

# Install
./istioctl manifest apply -f ./istio.yaml

## Set up the base namespaces
kubectl apply -f ./namespaces.yaml

## Install Prometheus v2.28.1. The `helm upgrade -i` command can be used to
## update/redeploy as you change settings. Modify `prometheus-values.yaml`
## to change versions.
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade -i -n monitoring -f ./prometheus-values.yaml prometheus prometheus-community/prometheus
```

When this is done you'll have Prometheus deployed to the `monitoring` namespace and it'll be scraping using mTLS for pods that Istio has wrapped in an Envoy proxy.

Now deploy some app that can be scraped into the `test-app` namespace. Istio will automatically inject an Envoy sidecar and proxy it with permissive mTLS - that is, it should negotiate down to HTTP if the request doesn't present the right mTLS certificates.

On the `targets` page, look for the job `kubernetes-pods-istio-secure` - this is the job that scrapes the mTLS enabled pods.

- Using v2.28.1, the scrape will fail with a `connection reset by peer` error.
- Using v2.20.1, the scrape succeeds.
