# `type: LoadBalancer `

On cloud providers which support external load balancers, setting the **`type`** field to **`LoadBalancer`** provisions a load balancer for your Service. The actual creation of the load balancer happens asynchronously, and information about the provisioned balancer is published in the Service's **`.status.loadBalancer`** field. For example:

## Example

**`nginx.yml`**

```yml
# load balancer svc
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
---
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
```

```bash
kubectl apply -f nginx.yml
kubectl get pods -o wide
kubectl get svc -o wide # nginx-loadbalancer-service status: Pending
```

## MetalLB

Kubernetes does not offer an implementation of network load balancers (Services of type LoadBalancer) for bare-metal clusters. The implementations of network load balancers that Kubernetes does ship with are all glue code that calls out to various IaaS platforms (GCP, AWS, Azure…). If you’re not running on a supported IaaS platform (GCP, AWS, Azure…), LoadBalancers will remain in the **`pending`** state indefinitely when created.

Bare-metal cluster operators are left with two lesser tools to bring user traffic into their clusters,**`NodePort`** and **`externalIPs`** services. Both of these options have significant downsides for production use, which makes bare-metal clusters second-class citizens in the Kubernetes ecosystem.

MetalLB aims to redress this imbalance by offering a network load balancer implementation that integrates with standard network equipment, so that external services on bare-metal clusters also “just work” as much as possible.

### Deploying Nginx With MetalLB In K8s

#### 1. Install MetalLB

MetalLB needs to be installed in Layer 2 mode.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.11/config/manifests/metallb-native.yaml
```

Verify installation:

```bash
kubectl get pods -n metallb-system
```

You should see controller and speaker pods running.

#### 2. Configure MetalLB

Create a configuration file **`metallb-config.yml`** with an **IP range**.

```yml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.6.210-192.168.6.219 # set your ip range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: my-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - metallb-ip-pool
```

Apply the configuration:

```bash
kubectl apply -f metallb-config.yml
```

#### 3. Deploy Nginx With LoadBalancer

Create a deployment and a service with a specific external IP.

```yml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
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
        image: nginx
        ports:
        - containerPort: 80
```

```yml
# nginx-svc-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
  loadBalancerIP: 192.168.6.210  # Assigning a specific IP from MetalLB's range
```

Apply the manifests:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-svc-loadbalancer.yaml
```

#### 4. Verify The Service

Check if the service has the correct external IP:

```bash
kubectl get svc nginx
```

Test access via curl:

```bash
curl http://192.168.6.210
```

You should see the default Nginx welcome page.

✅ MetalLB is now serving Nginx with the specified IP range! 🚀

![image](https://github.com/user-attachments/assets/a13769ed-07bd-43cf-bb4d-a3089da81ee5)
