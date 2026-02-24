# Project 2 — Zero-Downtime Blue/Green Deployment with Ansible

Deploy a **Dockerized app** to web servers using a **blue/green** strategy: run the new version on the inactive slot, health-check it, then switch Nginx to the new slot and roll back automatically if the post-switch health check fails. Deployments run **one host at a time** (`serial: 1`) for a rolling update across the group.

---

## Goal

- **Zero-downtime**: New container starts on the inactive port; traffic switches only after the new app is healthy.
- **Rollback on failure**: If the health check via Nginx fails after the switch, Nginx and the active-slot file are reverted and the failed container is removed.
- **Reusable**: Works with any inventory that provides a `role_web` group (e.g. dynamic EC2 inventory from Project 1).

---

## What This Project Does

| Component | Description |
|-----------|-------------|
| **Docker** | Installs Docker Engine (CE), adds `ubuntu` to the `docker` group. |
| **ECR login** | Logs Docker into AWS ECR so the host can pull the app image. |
| **Nginx** | Installs Nginx, configures a site that proxies to an upstream (blue or green port). |
| **Blue/Green app** | Manages two slots (blue=8000, green=8001). Pulls image, starts new version on inactive slot, health-checks, switches Nginx upstream, updates active-slot file; on failure, rolls back and fails the play. |

Variables (app name, ECR registry, ports, health path, image tag, etc.) come from `group_vars/all/main.yml` and optionally from `group_vars/all/vault.yml` (e.g. `vault_app_env` for container env vars).

---

## Project Structure

```
project-2-zero-downtime-deployment/
├── README.md
├── ansible/
│   ├── group_vars/
│   │   └── all/
│   │       ├── main.yml         # app_name, ECR, ports, nginx, health, image_tag
│   │       └── vault.yml        # Encrypted secrets (e.g. vault_app_env)
│   ├── playbooks/
│   │   └── deploy_bluegreen.yml # Blue/green deploy on role_web, serial 1
│   └── roles/
│       ├── docker/              # Install Docker, add ubuntu to docker group
│       ├── ecr_login/           # Install AWS CLI, docker login to ECR
│       ├── nginx/               # Install Nginx, upstream + site config
│       └── app_bluegreen/       # Blue/green slot logic, health checks, rollback
```

- **`deploy_bluegreen.yml`**: Targets `role_web`, loads `group_vars/all/main.yml` and `vault.yml`, runs roles: docker → ecr_login → nginx → app_bluegreen. Uses `serial: 1` for rolling deploy and `any_errors_fatal: true`.
- **`group_vars/all/main.yml`**: App name, AWS region, ECR registry/repo, image tag, blue/green ports, Nginx server name and listen port, health path and retry settings. Optional: `vault_app_env: {}` (override in vault for container env vars).
- **`group_vars/all/vault.yml`**: Vault-encrypted variables (e.g. `vault_app_env`) for sensitive container env; referenced in the playbook via `vars_files`.

---

## How It Works

### 1. Blue/Green Slots

- **Blue** = port 8000, **Green** = port 8001 (configurable).
- The **active** slot is the one Nginx proxies to; the **inactive** slot is where the new version is started.
- State is stored in `/opt/{{ app_name }}/active_slot` (e.g. `blue` or `green`). On first run, it defaults to blue.

### 2. Deploy Flow (per host)

1. **Ensure app dir** and read current `active_slot` (default blue).
2. **Compute** active/inactive ports and container names (`myapp-blue`, `myapp-green`).
3. **Pull** the target image (`image_full` from vars).
4. **Remove** the inactive container if it exists (safe to replace).
5. **Start** the new image on the inactive port with `--restart unless-stopped` and optional env from `vault_app_env`.
6. **Health check** the candidate on `http://127.0.0.1:{{ inactive_port }}{{ health_path }}` (retries/delay from vars).
7. **Switch**: Write Nginx upstream to the candidate port, reload Nginx, update `active_slot` file.
8. **Reload** Nginx (immediate), then **stop** the old active container (cleanup).
9. **Final health check** via Nginx on `http://127.0.0.1:{{ nginx_listen_port }}{{ health_path }}`. If it fails, **rescue** runs: revert upstream and slot file, flush handlers, remove failed container, then **fail** the play with a rollback message.

### 3. Rollback

If the final health check (via Nginx) fails after the switch, the play enters **rescue**: Nginx upstream and the active-slot file are reverted to the previous active port/slot, handlers are flushed (so Nginx reload runs), the failed candidate container is removed, and the play then fails with a clear message. The host is left serving the previous version.

### 4. Serial and Inventory

- The playbook uses **serial: 1** so only one host in `role_web` is updated at a time (rolling deploy).
- **Inventory**: Expects a group **role_web** (e.g. from Project 1’s dynamic inventory `aws_ec2` keyed by `tags.Role`). No inventory is included in this project; use your own (e.g. `-i inventories/dev/aws_ec2.yml` from Project 1).

---

## Requirements

- **Ansible** (e.g. 2.14+).
- **Target hosts**: Ubuntu (or similar) with sudo; able to install Docker and Nginx.
- **AWS**: ECR repository and image; hosts need **AWS credentials** (e.g. IAM instance profile or env) so `aws ecr get-login-password` and `docker pull` work.
- **App image**: Must listen on `container_internal_port` (e.g. 8000) and expose a **health endpoint** at `health_path` (e.g. `/health`) returning 200.

---

## Usage

### Configure variables

- Edit **`ansible/group_vars/all/main.yml`**: Set `app_name`, `aws_region`, `ecr_registry`, `ecr_repo`, ports, `health_path`, `nginx_listen_port`, etc. Set `image_tag` or pass it at run time.
- Optional: Put container env in **vault**:
  ```bash
  ansible-vault edit ansible/group_vars/all/vault.yml
  ```
  Add e.g.:
  ```yaml
  vault_app_env:
    NODE_ENV: production
    SOME_SECRET: "{{ some_vault_var }}"
  ```

### Run the playbook

From the **ansible** directory (so `vars_files` paths resolve correctly), with an inventory that defines **role_web**:

```bash
cd ansible
ansible-playbook playbooks/deploy_bluegreen.yml -i /path/to/inventory \
  --ask-vault-pass
# Or pass image tag and/or vault password file:
ansible-playbook playbooks/deploy_bluegreen.yml -i inventories/dev/aws_ec2.yml \
  -e image_tag=v1.2.3 --vault-password-file=~/.vault_pass
```

If using Project 1’s dynamic inventory (after provisioning):

```bash
cd project-2-zero-downtime-deployment/ansible
ansible-playbook playbooks/deploy_bluegreen.yml -i ../../project-1-aws-infra-provisioning/ansible/inventories/dev/aws_ec2.yml \
  -e image_tag=latest --ask-vault-pass
```

---

## Idempotency

- **Docker / Nginx / ECR login**: Install and config tasks are idempotent; re-runs are safe.
- **Blue/Green**: Each run deploys the requested image to the inactive slot and switches traffic. Re-running with the same image is safe (same version runs on both slots until the next deploy).

---

## Security Notes

- **Secrets**: Keep ECR credentials and app secrets out of the repo; use **Ansible Vault** for `vault_app_env` and any sensitive vars.
- **ECR**: Hosts must have IAM permissions for `ecr:GetAuthorizationToken` and (if private) `ecr:BatchGetImage` / `ecr:GetDownloadUrlForLayer` for the repo.
- **Nginx**: Listens on `nginx_listen_port` (default 80); restrict access with security groups or a reverse proxy (e.g. ALB) as in your Project 1 setup.

---

## What You Learn

- **Blue/green deployments**: Inactive slot, health check, then switch; automatic rollback on failure.
- **Ansible block/rescue**: Using rescue to revert and then fail after a failed health check.
- **Nginx as local reverse proxy**: Single upstream that points to either blue or green port.
- **Docker + ECR**: Installing Docker, logging into ECR, and running containers with env from Vault.
- **Rolling updates**: `serial: 1` and targeting `role_web` for one-at-a-time deploys across the group.

---

## References

- [Ansible block/rescue](https://docs.ansible.com/ansible/latest/user_guide/playbooks_blocks.html#handling-failure-with-blocks)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
