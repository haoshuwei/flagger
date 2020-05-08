# Flagger Install on Alibaba Cloud Service Mesh

This guide walks you through setting up Flagger on Alibaba Cloud Service Mesh.


## Prerequisites

You will be creating a instance on Alibaba Cloud Service Mesh \(ASM\) and several clusters on Alibaba Cloud Container Service for Kubernetes \(ACK\), then join clusters into a asm instance. If you don’t have an account you can sign up [here](https://account.aliyun.com/register/register.htm) for free credits.

## Create Cluster(s) on Alibaba Cloud Container Service for Kubernetes
Login into [Alibaba Cloud Container Service for Kubernetes](https://cs.console.aliyun.com/), create one or more clusters.
You can follow [creating ack clusters](https://www.alibabacloud.com/help/doc-detail/86488.htm) to create a kubernetes cluster.

## Create ASM instance on Alibaba Cloud Service Mesh
Login into [Alibaba Cloud Service Mesh](https://servicemesh.console.aliyun.com/), create an asm instance.
You can follow [creating asm instance](https://help.aliyun.com/document_detail/152154.html) to create an instance.

## Add ACK Clusters into an ASM Instance
You can follow [add clusters into asm](https://help.aliyun.com/document_detail/150509.html) to add ack clusters into a asm instance.

Create public gateway on ASM:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

Save the above resource as public-gateway.yaml and then apply it with ASM instance's kubeconfig:

```bash
kubectl apply -f ./public-gateway.yaml
```

## Install Prometheus

The ASM instance can help create a Prometheus instance. You can get the service URL by command:

```bash

```

## Install Flagger

Create a secret with the ASM instance's kubeconfig file:

```bash
kubectl -n istio-system create secret generic istio-kubeconfig --from-file kubeconfig
kubectl -n istio-system label secret istio-kubeconfig  istio/multiCluster=true
```

Add Flagger Helm repository:

```bash
helm repo add flagger https://flagger.app
```

Install Flagger's Canary CRD:

```yaml
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
```

Deploy Flagger in the `istio-system` namespace with Slack notifications enabled:

```bash
helm upgrade -i flagger flagger/flagger \
--namespace=istio-system \
--set crd.create=false \
--set metricsServer=http://prometheus.istio-system:9090 \
--set meshProvider=istio
--set istio.kubeconfig.secretName=istio-kubeconfig
--set istio.kubeconfig.key=kubeconfig
--set slack.url=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK \
--set slack.channel=general \
--set slack.user=flagger
```

## Install Grafana

Deploy Grafana in the `istio-system` namespace:

```bash
helm upgrade -i flagger-grafana flagger/grafana \
--namespace=istio-system \
--set url=http://prometheus.istio-system:9090 \
--set user=admin \
--set password=replace-me
```

Expose Grafana through the public gateway by creating a virtual service \(replace `example.com` with your domain\):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana
  namespace: istio-system
spec:
  hosts:
    - "grafana.istio.example.com"
  gateways:
    - public-gateway.istio-system.svc.cluster.local
  http:
    - route:
        - destination:
            host: flagger-grafana
```

Save the above resource as grafana-virtual-service.yaml and then apply it with ASM instance's kubeconfig:

```bash
kubectl apply -f ./grafana-virtual-service.yaml
```

Navigate to `http://grafana.istio.example.com` in your browser.

