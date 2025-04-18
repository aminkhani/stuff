# Minikube

![minikube](minikube.png)

- local single-node cluster using Minikube.
- Local clusters are the most useful for developers that want quick
edit-test-deploy-debug cycles on their machine, before committing their changes

## Installation

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

```bash
minikube start
```

```bash
kubectl get pod -A
```

### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
