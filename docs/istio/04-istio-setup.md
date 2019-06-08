# Install Istio with Helm

Download the latest Istio release:

```bash
curl -L https://git.io/getLatestIstio | sh -
```

Navigate to `istio-x.x.x` dir and copy the Istio CLI in your bin:

```bash
cd istio-1.0.3/
sudo cp ./bin/istioctl /usr/local/bin/istioctl
```

Apply the Istio CRDs:

```bash
kubectl apply -f ./install/kubernetes/helm/istio/templates/crds.yaml
```

> As described in https://github.com/istio/istio/issues/10588#issuecomment-466775037 this is no longer required and will raise ``Error: customresourcedefinitions.apiextensions.k8s.io "adapters.config.istio.io" already exists`` on ``helm install/upgrade``

Create a service account and a cluster role binding for Tiller:

```bash
kubectl apply -f ./install/kubernetes/helm/helm-service-account.yaml
```

Deploy Tiller in the `kube-system` namespace:

```bash
helm init --service-account tiller
```

Find the GKE IP ranges:

```bash
gcloud container clusters describe istio --zone=europe-west3-a \
| grep -e clusterIpv4Cidr -e servicesIpv4Cidr
```

You'll be using the IP ranges to allow unrestricted egress traffic for services running inside the service mesh.

Configure Istio with Prometheus, Jaeger, and cert-manager:

```yaml
global:
  nodePort: false
  proxy:
    # replace with your GKE IP ranges
    includeIPRanges: "10.28.0.0/14,10.7.240.0/20"

sidecarInjectorWebhook:
  enabled: true
  enableNamespacesByDefault: false

gateways:
  enabled: true
  istio-ingressgateway:
    replicaCount: 2
    autoscaleMin: 2
    autoscaleMax: 3
    # replace with your Istio Gateway IP
    loadBalancerIP: "35.198.98.90"
    type: LoadBalancer

pilot:
  enabled: true
  replicaCount: 1
  autoscaleMin: 1
  autoscaleMax: 1
  resources:
    requests:
      cpu: 500m
      memory: 1024Mi

grafana:
  enabled: true
  security:
    enabled: true
    adminUser: admin
    # change the password
    adminPassword: admin

prometheus:
  enabled: true

servicegraph:
  enabled: true

tracing:
  enabled: true
  jaeger:
    tag: 1.7

certmanager:
  enabled: true
```

Save the above file as `my-istio.yaml` and install Istio with Helm:

```bash
helm upgrade --install istio ./install/kubernetes/helm/istio \
--namespace=istio-system \
-f ./my-istio.yaml
```

Verify that Istio workloads are running:

```bash
kubectl -n istio-system get pods
```

Next: [Configure Istio Gateway with Let's Encrypt wildcard certificate](05-letsencrypt-setup.md)
