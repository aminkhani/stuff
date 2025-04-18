# Deployments

A Deployment provides declarative updates for **`Pods`** and **`ReplicaSets`**.

You describe a *desired state* in a Deployment, and the Deployment **Controller** changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.

## Use Case

The following are typical use cases for Deployments:

- **Create** a Deployment to rollout a ReplicaSet. The ReplicaSet creates Pods in the background. Check the status of the rollout to see if it succeeds or not.
- **Declare** the new state of the Pods by updating the PodTemplateSpec of the Deployment. A new ReplicaSet is created and the Deployment manages moving the Pods from the old ReplicaSet to the new one at a controlled rate. Each new ReplicaSet updates the revision of the Deployment.
- **Rollback** to an earlier Deployment revision if the current state of the Deployment is not stable. Each rollback updates the revision of the Deployment.
- **Scale up** the Deployment to facilitate more load.
- **Pause** the rollout of a Deployment to apply multiple fixes to its PodTemplateSpec and then resume it to start a new rollout.
- Use the status of the Deployment as an indicator that a rollout has stuck.
- **Clean up** older ReplicaSets that you don't need anymore.

### Create a Nginx Deployment With 3 Replicas

**[nginx-deployment.yml](/configs/deployments/nginx-deployment.yml)**

```bash
kubectl apply -f nginx-deployment.yml
```

### Rollout To Previous Version

Make a change to your manifest (`replicas: 3 -> 2`).

In these scenario im going to add a label named **`test=a`** in my manifest (it can be anything you want)

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  strategy:
    type: Recreate
  revisionHistoryLimit: 100
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        test: a
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Create Nodeport Service with fixed port 3007

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc # create a service with "nginx" name
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80 # By default and for convenience, the `targetPort` is set to
      # the same value as the `port` field.
      targetPort: 80 # Optional field
      # By default and for convenience, the Kubernetes control plane
      # will allocate a port from a range (default: 30000-32767)
      nodePort: 30007
  selector: # headless service provides to reach pod with podName.serviceName
    app: nginx
```

Then you can see the list of version that your deployment has and revert your change to previoius version

```bash
kubectl rollout history deployment/nginx-deployment

kubectl rollout undo deployment/nginx-deployment --to-revision=2
```
