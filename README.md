# Project 1 — Production-Grade AWS Infrastructure with Ansible

Provision a full **VPC-based environment** in AWS using Ansible: custom VPC, public/private subnets, Internet Gateway, NAT Gateway, security groups, and EC2 instances (bastion, web servers, database). All resources are tagged, secrets live in Ansible Vault, and SSH to private hosts goes through the bastion.

---

## Goal

- **Infrastructure as Code**: Define and manage AWS networking and compute with Ansible.
- **Security**: Private subnets for app/DB; SSH only via bastion; secrets in Vault.
- **Operability**: Dynamic inventory from EC2, separate dev/prod layout, idempotent runs, clean teardown.

---

## What This Project Builds

| Component | Description |
|-----------|-------------|
| **Custom VPC** | Single VPC with a configurable CIDR (e.g. `10.20.0.0/16`). |
| **2 Public subnets** | One per AZ; used for bastion and NAT Gateway. |
| **2 Private subnets** | One per AZ; used for web and DB instances. |
| **Internet Gateway** | Attached to the VPC for public internet access. |
| **NAT Gateway** | In the first public subnet; gives private instances outbound internet. |
| **Route tables** | Public: default route via IGW. Private: default route via NAT. |
| **Security groups** | Bastion (SSH from internet), Web (SSH from bastion + HTTP), DB (SSH from bastion + PostgreSQL from web). |
| **EC2 instances** | 1 bastion (public), 2 web servers (private), 1 DB server (private). |

All resources are tagged with `Project`, `Environment`, and `ManagedBy: Ansible` (plus `Role` and `Name` where relevant) so you can target them with dynamic inventory and teardown.

---

## Project Structure

```
project-1-aws-infra-provisioning/
├── README.md
├── ansible/
│   ├── ansible.cfg              # Inventory path, callback, forks, inventory plugin
│   ├── group_vars/
│   │   └── all/
│   │       ├── main.yml         # Region, VPC/subnet CIDRs, instance types, tags, SSH key
│   │       └── vault.yml        # Encrypted secrets (Ansible Vault)
│   ├── inventories/
│   │   ├── dev/
│   │   │   ├── aws_ec2.yml      # Dynamic inventory config for dev (aws_ec2 plugin)
│   │   │   └── group_vars/
│   │   │       └── all.yml     # Bastion IP, ProxyJump for SSH
│   │   └── prod/               # Same layout for prod (group_vars, aws_ec2.yml)
│   ├── playbooks/
│   │   ├── provision.yml        # Create VPC, subnets, IGW, NAT, SGs, EC2s
│   │   └── teardown.yml         # Terminate instances, then delete VPC
│   └── roles/                  # (Optional; current playbooks are task-based)
```

- **`ansible.cfg`**: Sets default inventory to `inventories/dev/aws_ec2.yml`, enables the `amazon.aws.aws_ec2` inventory plugin, and tunes SSH/forks/callback.
- **`group_vars/all/main.yml`**: Single source for region, project name, env name, VPC/subnet definitions, instance types, AMI, SSH key name, and common tags.
- **`group_vars/all/vault.yml`**: Vault-encrypted variables (e.g. any sensitive overrides); referenced in playbooks via `vars_files`.
- **Inventories**: Dev (and prod when added) use a YAML file that configures the **aws_ec2** dynamic inventory plugin (regions, filters, keyed_groups, hostnames, compose). This discovers EC2s by tags (`Project`, `Environment`, `Role`) so you get groups like `role_bastion`, `role_web`, `role_db`, `env_dev`.
- **SSH**: Inventory `group_vars` set `ansible_ssh_common_args` with **ProxyJump** via the bastion; `bastion_public_ip` is written by the provision playbook after the bastion is created.

---

## How It Works

### 1. Provisioning (`provision.yml`)

The playbook runs on **localhost** with `connection: local` and uses the **amazon.aws** collection to talk to AWS. It:

1. **VPC & networking**: Creates VPC → IGW → public/private subnets → EIP for NAT → NAT Gateway → public route table (0.0.0.0/0 → IGW) → private route table (0.0.0.0/0 → NAT).
2. **Security groups**: Bastion (SSH 22 from 0.0.0.0/0), Web (SSH from bastion SG, HTTP 80 from 0.0.0.0/0), DB (SSH from bastion SG, PostgreSQL 5432 from web SG).
3. **SSH key**: Ensures an EC2 key pair exists; if newly created, saves the private key locally.
4. **AMI**: Looks up the latest Ubuntu 22.04 LTS AMI in the target region.
5. **EC2 instances**: Launches bastion in the first public subnet (public IP), two web servers in the first two private subnets, one DB server in the first private subnet. All are tagged with `Project`, `Environment`, and `Role` (bastion/web/db).
6. **Post-provision**: Prints the bastion public IP and updates `inventories/dev/group_vars/all.yml` with `bastion_public_ip` so subsequent Ansible runs use ProxyJump correctly.

Variables (region, CIDRs, instance types, tags, key name, etc.) come from `group_vars/all/main.yml` and optionally from `group_vars/all/vault.yml`.

### 2. Dynamic Inventory (`aws_ec2`)

- The **amazon.aws.aws_ec2** plugin is configured in `inventories/dev/aws_ec2.yml`: regions, `keyed_groups` (e.g. by `tags.Role`, `tags.Environment`), **filters** (`tag:Project`, `tag:Environment`), **hostnames** (e.g. private IP, DNS name), and **compose** (e.g. `ansible_user: ubuntu`, `ansible_host: private_ip_address`).
- After provisioning, `ansible-inventory -i inventories/dev/aws_ec2.yml --graph` (or `--list`) shows hosts and groups. Playbooks can target `role_web`, `role_db`, etc., and SSH to private hosts uses the bastion via ProxyJump.

### 3. SSH Only via Bastion

- Bastion is the only host with a public IP. Private hosts use `ansible_host: private_ip_address`.
- Inventory sets `ansible_ssh_common_args` with `ProxyJump=ubuntu@{{ bastion_public_ip }}`. So from your machine you SSH to the bastion, and Ansible uses the bastion to reach web/DB hosts.

### 4. Teardown (`teardown.yml`)

- Runs on localhost, uses `group_vars/all/main.yml` (same `project_name`, `env_name`, `aws_region`).
- **First**: Terminates all EC2 instances with matching `Project` and `Environment` tags (`amazon.aws.ec2_instance` with `state: absent` and filters).
- **Then**: Finds the VPC by the same tags and deletes it (`state: absent`). Dependent resources (subnets, route tables, SGs, NAT, IGW) are removed as part of VPC deletion where AWS allows. If dependencies remain, you may need to delete them (e.g. NAT Gateway, then EIP) before the VPC can be removed; the playbook is designed to be extended for full cleanup.

---

## Requirements

- **Ansible** (e.g. 2.14+).
- **Python 3** with **boto3** and **botocore** (for AWS modules and inventory plugin).
- **Ansible collection**: `amazon.aws` (e.g. `ansible-galaxy collection install amazon.aws`).
- **AWS credentials**: Configured via environment (`AWS_PROFILE`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`, etc.) or IAM role if the controller runs in AWS. No keys in playbooks; use Vault for any secret vars.

---

## Usage

### Install collection and Python deps

```bash
ansible-galaxy collection install amazon.aws
pip install boto3 botocore   # or use a venv and omit --user
```

### Configure environment

- Set **AWS_PROFILE** (or use default credentials):
  ```bash
  export AWS_PROFILE=your-profile
  ```
- Optional: Use an **IAM role** (e.g. on an EC2 controller) instead of access keys — no playbook changes needed; the collection uses the standard credential chain.

### Encrypt secrets (optional)

If you store secrets in `group_vars/all/vault.yml`:

```bash
ansible-vault create ansible/group_vars/all/vault.yml   # or edit, rekey
ansible-vault edit ansible/group_vars/all/vault.yml
```

Run playbooks with:

```bash
ansible-playbook ansible/playbooks/provision.yml --ask-vault-pass
# or: --vault-password-file=~/.vault_pass
```

### Provision

From the project root (or from `ansible/` and adjust paths):

```bash
cd ansible
ansible-playbook playbooks/provision.yml -e @group_vars/all/main.yml
# If using vault:
ansible-playbook playbooks/provision.yml --ask-vault-pass
```

After provisioning, ensure `inventories/dev/group_vars/all.yml` has `bastion_public_ip` set (the playbook writes it). Then list dynamic inventory:

```bash
ansible-inventory -i inventories/dev/aws_ec2.yml --graph
```

### Run ad-hoc or playbooks against EC2s

```bash
ansible role_web -i inventories/dev/aws_ec2.yml -m ping
ansible-playbook your_playbook.yml -i inventories/dev/aws_ec2.yml
```

### Teardown

```bash
ansible-playbook playbooks/teardown.yml
```

This terminates tagged instances and then deletes the tagged VPC. If AWS reports dependency errors, delete NAT Gateway (and release the EIP) first, then run teardown again.

---

## Idempotency

- **amazon.aws** modules are used with `state: present` or `state: absent` and identify resources by name, ID, or tags. Re-running **provision** creates only what is missing (e.g. existing VPC/subnets are not duplicated if names/tags match).
- **Teardown** is keyed by tags; running it multiple times is safe once instances are gone and VPC dependencies are cleared.

---

## Separate Dev/Prod Inventories

- **Dev**: `inventories/dev/aws_ec2.yml` and `inventories/dev/group_vars/all.yml`; `ansible.cfg` points at `inventories/dev/aws_ec2.yml` by default.
- **Prod**: Duplicate the layout under `inventories/prod/`: `aws_ec2.yml` (filters `Environment: prod`, same structure) and `group_vars/all.yml`. Use a different `project_name`/`env_name` or vars in `group_vars/all/main.yml` (or prod-specific group_vars) so provision/teardown and inventory target prod. Switch inventory with `-i inventories/prod/aws_ec2.yml` or a second `ansible.cfg`/environment.

---

## Advanced: IAM Role Instead of Access Keys

Run the Ansible controller on an EC2 instance (or elsewhere in AWS) and attach an IAM role with permissions for EC2, VPC, and (if used) SSM. Do not set `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`; boto3/Ansible will use the instance metadata (or IRSA in EKS). The same playbooks and inventory work without change.

---

## Security Notes

- **Bastion SG**: SSH (22) is open from `0.0.0.0/0`; in production, restrict to your IP or a VPN CIDR.
- **Secrets**: Keep AWS keys and other secrets out of repo; use **Ansible Vault** for any sensitive variables.
- **SSH**: Private keys are created by the playbook and stored locally; restrict permissions (e.g. `~/.ssh` 0700, key file 0600).

---

## What You Learn

- **VPC networking**: Subnets, IGW, NAT, route tables, and how traffic flows public vs private.
- **AWS modules in Ansible**: Using `amazon.aws` for VPC, subnets, security groups, key pairs, and EC2.
- **Infrastructure as Code**: Single source of truth in YAML; parameterized by vars and env-specific inventory.
- **Security practices**: Private subnets, minimal SGs, SSH via bastion, secrets in Vault.
- **Operability**: Dynamic inventory, tagging, idempotent provisioning, and a dedicated teardown path for clean destroy.

---

## References

- [Ansible AWS EC2 dynamic inventory guide](https://docs.ansible.com/ansible/latest/collections/amazon/aws/docsite/aws_ec2_guide.html)
- [amazon.aws collection](https://docs.ansible.com/ansible/latest/collections/amazon/aws/index.html)
