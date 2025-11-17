# Envoy Gateway on Oracle Kubernetes Engine (OKE)

This repo contains a minimal, working example of running Envoy Gateway with Gateway API
on Oracle Kubernetes Engine (OKE), fronted by an OCI Load Balancer.

## Structure

- `manifests/`
  - `envoyproxy-oci.yaml` - Configures Envoy Gateway to expose an OCI Load Balancer with flexible shape.
  - `gatewayclass.yaml`   - GatewayClass that binds to the EnvoyProxy.
  - `gateway.yaml`        - Public HTTP Gateway in the `default` namespace.
  - `demo-app.yaml`       - Simple http-echo demo application and Service.
  - `httproute.yaml`      - HTTPRoute that sends all traffic (`/`) to `my-app`.

## Prerequisites

- An OKE cluster (1.30+ recommended).
- `kubectl` configured against the OKE cluster.
- `helm` installed locally.

## 1. Install Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.0 \
  -n envoy-gateway-system \
  --create-namespace
```

Wait for Envoy Gateway to be ready:

```bash
kubectl -n envoy-gateway-system get pods
```

## 2. Apply manifests

```bash
kubectl apply -k manifests/
```

## 3. Get the external IP

```bash
kubectl get svc -A | grep envoy
```

Take the `EXTERNAL-IP` of the Envoy Gateway LoadBalancer service and test:

```bash
curl http://<EXTERNAL-IP>/
```

You should see:

```text
Hello from Envoy Gateway on OKE
```

## 4. Cleanup

```bash
kubectl delete -k manifests/
helm uninstall eg -n envoy-gateway-system
kubectl delete namespace envoy-gateway-system
```