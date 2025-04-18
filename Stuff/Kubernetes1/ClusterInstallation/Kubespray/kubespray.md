# Kubernetes Installation with Kubespray

## Method 1: Kubespray (Local Installation)

### Prerequisites

```bash
# write your Kubernetes nodes ip :
# 192.168.7.192
# 192.168.7.193
# 192.168.7.197
```

### 1. Clone Kubespray

```bash
mkdir kubernetes_installation/
cd kubernetes_installation/
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray/
```

### 2. install requirements for kubespray

```bash
apt update && apt install python3-pip ansible
pip install -r requirements.txt  --break-system-packages
pip3 install ruamel.yaml --break-system-packages
# Run it on ALL Nodes
apt install sshpass -y
```

### 3. Determine Your Kubernetes nodes IP

```bash
cp -rfp inventory/sample inventory/mycluster
declare -a IPS=(192.168.7.192 192.168.7.193 192.168.7.197)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

You will be asked for ssh pass and sudo pass

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --user geek --become -kK cluster.yml
```

## Method 2: Kubespray (Docker Image)

### Prerequisites

- 3 Linux nodes with SSH access.
- Python and Ansible installed.
- Docker installed on the control node.
- Public and private SSH keys configured for passwordless SSH access.

### Inventory File Format

```ini
[kube_control_plane]
node1 ansible_host=192.168.6.130 ip=192.168.6.130 etcd_member_name=etcd1
node2 ansible_host=192.168.6.131 ip=192.168.6.131 etcd_member_name=etcd2
node3 ansible_host=192.168.6.132 ip=192.168.6.132 etcd_member_name=etcd3

[etcd:children]
kube_control_plane

[kube_node]
node1 ansible_host=192.168.6.130 ip=192.168.6.130
node2 ansible_host=192.168.6.131 ip=192.168.6.131
node3 ansible_host=192.168.6.132 ip=192.168.6.132

[k8s_cluster:children]
kube_control_plane
kube_node

```

**Steps to Install Kubernetes**

### 1. Clone Kubespray Repository

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

### 2. Create Inventory File

```bash
cp -rfp inventory/sample inventory/mycluster
vi inventory/mycluster/hosts.ini
```

Paste the inventory format provided above and modify as per your setup.

### 3. Run Kubespray Using Docker

```bash
docker run --rm -it \
  -v $PWD:/kubespray \
  -v ~/.ssh:/root/.ssh \
  -e ANSIBLE_CONFIG=/kubespray/ansible.cfg \
  quay.io/kubespray/kubespray:v2.23.0 \
  ansible-playbook -i /kubespray/inventory/mycluster/hosts.ini --become --become-user=root cluster.yml
```
