# TLS - expand Gateway to support TLS
## Will use Let'sencrypt and nip.io

---

## 📌 Prerequisites

* A working **Envoy Gateway** installation on OKE
* An `EnvoyProxy` object that creates a **LoadBalancer Service**
* A valid **public IP address** for the Envoy LB
* A functional **Gateway** and **HTTPRoute**
* A deployed backend app (e.g. `demo-app`)

---

## 🚀 Install cert-manager (Simple Helm Method)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --set "extraArgs={--enable-gateway-api}"
```

```bash
kubectl get pods -n cert-manager
```

---

## 🌐 1. Choose a Domain

Use a free dynamic DNS via **nip.io**:

```
echo.<LB_IP>.nip.io
```

Example:

```
echo.79.76.98.111.nip.io
```

---

## 🔐 2. Create a ClusterIssuer (Let’s Encrypt)

Apply:

```bash
kubectl apply -f manifests/tls/clusterissuer.yaml
```

---

## 📄 3. Request a Certificate

Apply:

```bash
kubectl apply -f manifests/tls/certificate.yaml
```

Check status:

```bash
kubectl describe certificate echo-tls -n default
kubectl get challenges -A
```

---

## 🌉 4. Add HTTPS Listener to the Gateway

Apply:

```bash
kubectl apply -f gateway-https.yaml
```

---

## 🌉 5. Create HTTP Route bound to the TLS hostname

Apply:

```bash
kubectl apply -f manifests/tls/route_tls.yaml
```

---

## 🧪 6. Test HTTPS

```bash
curl -v https://echo.79.76.98.111.nip.io/
```