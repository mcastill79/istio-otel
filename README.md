# Enable New Relic Observability of Istio Metrics and Traces with Otel

This repo contains an example of the assets required to deploy an otel operator, a target allocator and the otel collector, 
to scrape istio metrics, process Istio traces and export both to New Relic. 

```
Some of the sections here are as a reference only (i.e. Installation of Minikube, Installation of Istio, 
Demo profile and sample application). They provide the information on the required Istio configuration.

Summary of the steps to implement the Otel collector:

1. kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.14.4/cert-manager.yaml
2. kubectl create ns otel-system
3. kubectl config set-context --current --namespace=otel-system
4. kubectl label namespace otel-system istio-injection=disabled
5. helm install my-opentelemetry-operator open-telemetry/opentelemetry-operator  -n otel-system
6. kubectl create secret generic nr-key --from-literal=NEW_RELIC_LICENSE_KEY=xxxxxxxx -n otel-system
7. cd collector
8. helm install otel-collector -f values.yaml . -n otel-system
```


### Install Minikube
```
$minikube start --cpus 4 --memory 4096
😄  minikube v1.32.0 on Darwin 14.3 (arm64)
✨  Using the podman (experimental) driver based on user configuration
📌  Using Podman driver with root privileges
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
E0320 09:36:24.304782   13186 cache.go:189] Error downloading kic artifacts:  not yet implemented, see issue #8426
🔥  Creating podman container (CPUs=4, Memory=4096MB) ...
🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7 ...E0320 09:36:38.932558   13186 start.go:131] Unable to get host IP: RoutableHostIPFromInside is currently only implemented for linux

    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass

❗  /Users/test-user/bin/kubectl is version 1.22.3, which may have incompatibilities with Kubernetes 1.28.3.
    ▪ Want kubectl v1.28.3? Try 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

```

## Install Istio with demo profile and sample application

```
$curl -L https://istio.io/downloadIstio | sh -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   101  100   101    0     0    594      0 --:--:-- --:--:-- --:--:--   597
100  4899  100  4899    0     0  16711      0 --:--:-- --:--:-- --:--:-- 16711

Downloading istio-1.21.0 from https://github.com/istio/istio/releases/download/1.21.0/istio-1.21.0-osx-arm64.tar.gz ...

Istio 1.21.0 Download Complete!

Istio has been successfully downloaded into the istio-1.21.0 folder on your system.

Next Steps:
See https://istio.io/latest/docs/setup/install/ to add Istio to your Kubernetes cluster.

To configure the istioctl client tool for your workstation,
add the /Users/test-user/github/nr/otel/istio-1.21.0/bin directory to your environment path variable with:
	 export PATH="$PATH:/Users/test-user/github/nr/otel/istio-1.21.0/bin"

Begin the Istio pre-installation check by running:
	 istioctl x precheck

Need more information? Visit https://istio.io/latest/docs/setup/install/
$./istio-1.21.0/bin/istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete                                                                                                                                                                                                                     Made this installation the default for injection and validation.
$k get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-f6cc5c849-8shz7     1/1     Running   0          59s
istio-ingressgateway-76f54f4985-p928j   1/1     Running   0          59s
istiod-75685b54d6-jvrhl                 1/1     Running   0          71s

$kubectl label namespace default istio-injection=enabled
namespace/default labeled
$kubectl apply -f istio-1.21.0/samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
$k get pods -n default
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-698d88b-7t66p         2/2     Running   0          82s
productpage-v1-675fc69cf-6spvn   2/2     Running   0          82s
ratings-v1-6484c4d9bb-vsfkq      2/2     Running   0          82s
reviews-v1-5b5d6494f4-k7c4x      2/2     Running   0          82s
reviews-v2-5b667bcbf8-5nwdn      2/2     Running   0          82s
reviews-v3-5b9bd44f4-7l2bf       2/2     Running   0          82s
$kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>

```

### Install cert-manager ( dependency to run otel target allocator+collector )

```
$kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.14.4/cert-manager.yaml
namespace/cert-manager created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrole.rbac.authorization.k8s.io/cert-manager-cluster-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificatesigningrequests created
clusterrole.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-approve:cert-manager-io created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificatesigningrequests created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-webhook:subjectaccessreviews created
role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
role.rbac.authorization.k8s.io/cert-manager:leaderelection created
role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
service/cert-manager created
service/cert-manager-webhook created
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created

$kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-67c98b89c8-k5n2q              1/1     Running   0          34s
cert-manager-cainjector-5c5695d979-59wp7   1/1     Running   0          34s
cert-manager-webhook-7f9f8648b9-44j5q      1/1     Running   0          34s

```

### Install otel operator and put everything in otel-system ns

```
$kubectl create ns otel-system
namespace/otel-system created

$kubectl label namespace otel-system istio-injection=disabled
namespace/otel-system labeled

$helm install my-opentelemetry-operator open-telemetry/opentelemetry-operator  -n otel-system
NAME: my-opentelemetry-operator
LAST DEPLOYED: Wed Mar 20 09:46:49 2024
NAMESPACE: otel-system
STATUS: deployed
REVISION: 1
NOTES:
opentelemetry-operator has been installed. Check its status by running:
  kubectl --namespace otel-system get pods -l "release=my-opentelemetry-operator"

Visit https://github.com/open-telemetry/opentelemetry-operator for instructions on how to create & configure OpenTelemetryCollector and Instrumentation custom resources by using the Operator.

$kubectl get pods -n otel-system
NAME                                         READY   STATUS    RESTARTS   AGE
my-opentelemetry-operator-674cf94667-s76ts   2/2     Running   0          22s

$kubectl create secret generic nr-key --from-literal=NEW_RELIC_LICENSE_KEY=xxxxxxxxxxxxxxxxxx -n otel-system
secret/nr-key created

$cd collector/

$helm install otel-collector -f values.yaml . -n otel-system
NAME: otel-collector
LAST DEPLOYED: Wed Mar 20 09:51:21 2024
NAMESPACE: otel-system
STATUS: deployed
REVISION: 1
TEST SUITE: None

$kubectl get pods -n otel-system
NAME                                             READY   STATUS    RESTARTS   AGE
my-opentelemetry-operator-674cf94667-s76ts       2/2     Running   0          6m45s
otel-collector-collector-0                       1/1     Running   0          2m14s
otel-collector-targetallocator-cc55967b9-4ff9d   1/1     Running   0          2m14s

```
