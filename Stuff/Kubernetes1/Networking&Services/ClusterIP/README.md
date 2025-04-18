# `type: ClusterIP`

![ClusterIP](https://github.com/user-attachments/assets/0ed2a5d5-05f9-4f46-b5dd-b2cb5a187810)

In Kubernetes, the ClusterIP is a virtual IP (VIP) assigned to services to enable internal communication within the cluster. By default, Kubernetes will automatically assign a ClusterIP from the range defined in the cluster's configuration. However, in certain cases, you might want to set a specific IP for your ClusterIP service.

If you define a Service that has the **`.spec.clusterIP`** set to **`"None"`** then Kubernetes does not assign an IP address. See **[headless Services](../Headless/README.md)** for more information.

## Choosing your own IP address

You can specify your own cluster IP address as part of a **`Service`** creation request. To do this, set the **`.spec.clusterIP`** field. For example, if you already have an existing DNS entry that you wish to reuse, or legacy systems that are configured for a specific IP address and difficult to re-configure.

The IP address that you choose must be a valid IPv4 or IPv6 address from within the **`service-cluster-ip-range`** CIDR range that is configured for the API server. If you try to create a Service with an invalid clusterIP address value, the API server will return a 422 HTTP status code to indicate that there's a problem.

---

### Example

**`nginx.yml`**

```yml
# nginx deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginxp-clusterip-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```

```bash
kubectl apply -f nginx.yml
kubectl get pods -o wide
kubectl get svc -o wide
```
