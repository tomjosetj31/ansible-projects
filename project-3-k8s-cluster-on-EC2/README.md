# Project 3 — Kubernetes Cluster on EC2 with Ansible

Provision EC2 instances for a Kubernetes cluster and bootstrap them with **kubeadm**: one control-plane (master) node and multiple worker nodes, with **Calico** as the Pod network (CNI). All nodes get containerd, kubelet, kubeadm, and kubectl; the master is initialized and workers join using a generated token.

---

## Goal

- **Infrastructure**: Create EC2 instances tagged by role (`k8s_master`, `k8s_worker`) for use with dynamic inventory.
- **Bootstrap**: Install container runtime and Kubernetes packages on all nodes; disable swap.
- **Cluster**: Initialize the master with kubeadm, copy kubeconfig for `kubectl`, then join workers using a saved join command.
- **Networking**: Install Calico (CNI) so Pods get IPs and can communicate.

---

## What This Project Does

| Component | Description |
|-----------|-------------|
| **provision_k8s_nodes.yml** | Runs on localhost; creates 1 master and 2 worker EC2 instances in AWS (region, type, AMI, key name in vars). |
| **k8s_bootstrap.yml** | Runs on `role_k8s_master` and `role_k8s_worker`; installs containerd, kubelet, kubeadm, kubectl; disables swap. |
| **k8s_cluster.yml** | Three plays: (1) Initialize master with kubeadm, set up `~/.kube/config`, generate join command; (2) Join workers using that command; (3) Apply Calico manifest on the master. |
| **k8s_common** | Disable swap; install and enable containerd; install kubelet, kubeadm, kubectl. |
| **k8s_master** | `kubeadm init --pod-network-cidr=192.168.0.0/16`; copy admin.conf to ubuntu’s `.kube/config`; create join token and save as fact for workers. |
| **k8s_worker** | Run the join command from the master’s saved fact. |
| **k8s_network** | Copy Calico manifest to master and run `kubectl apply -f` (with `KUBECONFIG=/etc/kubernetes/admin.conf`). |

---

## Project Structure

```
project-3-k8s-cluster-on-EC2/
├── README.md
├── ansible/
│   ├── files/
│   │   └── k8s/
│   │       └── calico.yaml       # Calico manifest for CNI
│   ├── playbooks/
│   │   ├── provision_k8s_nodes.yml  # Create EC2 master + workers (localhost)
│   │   ├── k8s_bootstrap.yml        # Install deps on all K8s nodes
│   │   └── k8s_cluster.yml         # Init master, join workers, install Calico
│   └── roles/
│       ├── k8s_common/           # Swap off, containerd, kubelet/kubeadm/kubectl
│       ├── k8s_master/           # kubeadm init, kubeconfig, join token
│       ├── k8s_worker/           # kubeadm join using master’s token
│       └── k8s_network/         # Apply Calico on master
```

- **provision_k8s_nodes.yml**: Uses `amazon.aws.ec2_instance`; you must set a valid **AMI** and **key_name** (and optionally region, instance_type). Tags instances with `Role: k8s_master` / `Role: k8s_worker` for dynamic inventory (e.g. `aws_ec2` with keyed_groups by `tags.Role`).
- **k8s_bootstrap.yml**: Run once you have inventory groups `role_k8s_master` and `role_k8s_worker` (e.g. from the same EC2 inventory).
- **k8s_cluster.yml**: Run after bootstrap; order matters: master first (to produce join command), then workers, then CNI on master.

---

## How It Works

### 1. Provision EC2 nodes

Run `provision_k8s_nodes.yml` with `connection: local` and the **amazon.aws** collection. It creates:

- 1 instance named `k8s-master` with tag `Role: k8s_master`
- 2 instances named `k8s-worker` with tag `Role: k8s_worker`

Edit the playbook vars (or use `-e`) to set `region`, `instance_type`, `key_name`, and especially **`image_id`** (e.g. Ubuntu 22.04 AMI for your region). Ensure the AMI has cloud-init and standard repos so that containerd and Kubernetes packages can be installed.

### 2. Inventory

Use an inventory that resolves **role_k8s_master** and **role_k8s_worker**. For example:

- **Dynamic**: `amazon.aws.aws_ec2` inventory with `keyed_groups` on `tags.Role` so you get `role_k8s_master` and `role_k8s_worker`. Point the plugin at the same region and tag filters (e.g. by a project tag) so it only sees these instances.
- **Static**: Create an `inventories/` directory with a YAML or INI file that lists the master and workers in the two groups.

After provisioning, run `ansible-inventory -i your_inventory --graph` to confirm the groups.

### 3. Bootstrap (dependencies)

Run **k8s_bootstrap.yml** against `role_k8s_master:role_k8s_worker` so all nodes get:

- Swap disabled
- containerd installed and enabled
- kubelet, kubeadm, kubectl installed

Use the same inventory and SSH access (e.g. key, bastion) as for the rest of the playbooks.

### 4. Build the cluster (k8s_cluster.yml)

Run **k8s_cluster.yml**; it has three plays:

1. **Initialize master** (`role_k8s_master`): `kubeadm init --pod-network-cidr=192.168.0.0/16`, copy `/etc/kubernetes/admin.conf` to `/home/ubuntu/.kube/config`, create a join command and save it in a fact `k8s_join_command`.
2. **Join workers** (`role_k8s_worker`): Run the saved join command from the first master host (`hostvars[groups['role_k8s_master'][0]].k8s_join_command`).
3. **Install CNI** (`role_k8s_master`): Copy `files/k8s/calico.yaml` to the master and run `kubectl apply -f` with `KUBECONFIG=/etc/kubernetes/admin.conf` (delegate_to master, run_once).

After this, the cluster is ready: `kubectl get nodes` (from the master, as ubuntu) should show all nodes.

### 5. Calico

The **k8s_network** role expects the Calico manifest at `ansible/files/k8s/calico.yaml`. Use the official Calico manifest for your kubeadm/Kubernetes version (e.g. from [Calico docs](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)). The role copies it from the controller to the master and applies it there.

---

## Requirements

- **Ansible** (e.g. 2.14+).
- **Python 3** with **boto3** / **botocore** for EC2 (provision playbook).
- **Ansible collection**: `amazon.aws` (for `ec2_instance`).
- **AWS credentials**: Configured via profile, env, or IAM role so that the provision playbook can create EC2 instances and your inventory can discover them.
- **SSH**: Key pair named in `key_name` must exist in the region; targets (master/workers) must be reachable (e.g. security groups, bastion) so Ansible can run as `ubuntu` (or your user) with sudo.
- **Target OS**: Ubuntu (or similar) with standard repos; Kubernetes packages are installed via `apt`. For other distros you’d need to adjust k8s_common (and possibly package names in k8s_master/k8s_worker if any).

---

## Usage

### 1. Install collection

```bash
ansible-galaxy collection install amazon.aws
```

### 2. Provision EC2 nodes

Edit **region**, **image_id**, **key_name** (and optionally **instance_type**) in `provision_k8s_nodes.yml` or pass them with `-e`:

```bash
cd ansible
ansible-playbook playbooks/provision_k8s_nodes.yml \
  -e region=ap-south-1 \
  -e image_id=ami-0xxxxxxxx \
  -e key_name=mykey
```

### 3. Configure inventory

Ensure you have an inventory that defines **role_k8s_master** and **role_k8s_worker**. For dynamic inventory (e.g. `inventories/dev/aws_ec2.yml`), use the same filters/keyed_groups as in Project 1 so that EC2 instances with `Role: k8s_master` and `Role: k8s_worker` end up in those groups.

### 4. Bootstrap Kubernetes dependencies

```bash
ansible-playbook playbooks/k8s_bootstrap.yml -i inventories/dev/aws_ec2.yml
```

### 5. Initialize cluster and install CNI

Run from the **ansible** directory so `playbook_dir` resolves correctly for the Calico file path:

```bash
ansible-playbook playbooks/k8s_cluster.yml -i inventories/dev/aws_ec2.yml
```

### 6. Use kubectl

SSH to the master and run kubectl as ubuntu (kubeconfig is under `/home/ubuntu/.kube/config`):

```bash
ssh ubuntu@<master-ip>
kubectl get nodes
kubectl get pods -A
```

---

## Order of Operations

1. **provision_k8s_nodes.yml** — Create EC2 instances.
2. Wait for instances to be ready and inventory to see them (e.g. `ansible role_k8s_master -i inventories/dev/aws_ec2.yml -m ping`).
3. **k8s_bootstrap.yml** — Install containerd and Kubernetes packages on all nodes.
4. **k8s_cluster.yml** — Init master, join workers, apply Calico.

---

## Security Notes

- **EC2**: Restrict SSH (e.g. security group) to your IP or bastion; use a dedicated key for these instances.
- **Kubernetes**: The join token from `kubeadm token create` is short-lived; workers join in the same playbook run. For production, consider more restrictive token TTL and network policies.
- **Calico**: Use an official manifest that matches your Kubernetes version; review the manifest before applying.

---

## What You Learn

- **kubeadm**: Bootstrap a control-plane and join workers; pod-network-cidr and CNI.
- **Ansible and EC2**: Provision instances by role and use dynamic inventory to target master vs workers.
- **Cross-host facts**: Using `hostvars[groups['role_k8s_master'][0]].k8s_join_command` so workers get the join command from the master.
- **Delegate and run_once**: Applying a manifest once on the master with `delegate_to` and `run_once` in the k8s_network role.

---

## References

- [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [Calico on Kubernetes](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
- [amazon.aws.ec2_instance](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_instance_module.html)
