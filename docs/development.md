# Development

## Toolchain

- Terraform 0.13 or later with the Scaleway provider (locked at 2.2.7 in `plans/.terraform.lock.hcl`).
- Ansible on the control machine; `ansible.cfg` disables host key checking and enables pipelining.
- `jq` for `terraform_to_ansible_values.sh`.
- An SSH key accepted as `root` by the instances (Scaleway injects the keys of the project).

## Recipes

```sh
cd plans && terraform init && terraform plan                 # check the Scaleway plan
terraform apply                                              # create the three VMs
terraform output -json > terraform_values.json && cd ..      # outputs consumed by the script below
bash terraform_to_ansible_values.sh                          # fill inventories/scaleway.ini
ansible -i inventories/scaleway.ini k8s -m ping              # SSH access to the three hosts
ansible-playbook -i inventories/scaleway.ini set_local_hosts.yml --extra-vars @vars/scaleway.yml --ask-become-pass   # local /etc/hosts
ansible-playbook -i inventories/scaleway.ini playbooks/infrastructure.yml -t base,k8s-setup --extra-vars @vars/scaleway.yml   # several tags at once
ansible-playbook -i inventories/scaleway.ini playbooks/infrastructure.yml -t k8s-setup --extra-vars @vars/scaleway.yml --extra-vars "k8s_nodes_force_recreation=true k8s_nodes_force_rejoin=true"   # rebuild the cluster
ansible-playbook -i inventories/scaleway.ini playbooks/infrastructure.yml -t k8s-logging --extra-vars @vars/scaleway.yml   # one service only
KUBECONFIG=./config kubectl get nodes                         # from your machine, with the fetched admin.conf (master IP reachable on 6443)
cd plans && terraform destroy                                 # delete the VMs
```

The full ordered run is in [deployment.md](deployment.md).

## Reaching the cluster

- `ssh root@<master-ip>` then `kubectl`, `helm`, `linkerd` and `argocd` are on the PATH.
- Each UI is behind an ingress on `*.todevops.local`; the credentials are in `vars/scaleway.yml`.
- Prefer `kubectl port-forward` and an SSH tunnel for anything that is not a throwaway cluster,
  and delete the convenience ingresses (deployment.md, step 2.5).
- The Kubernetes dashboard has an `admin-user` service account bound to `cluster-admin`
  (`roles/setup_kubernetes_cp/files/dashboard-sa-crb.yml`); create a token for it with
  `kubectl -n kubernetes-dashboard create token admin-user`.

## Re-running and fixing

- Roles check the current state first (`helm status`, `kubectl get ...`) and skip what exists.
- `k8s-setup` resets kubeadm only when `kubectl get nodes` fails on the master or when
  `k8s_nodes_force_recreation` is true; nodes behave the same with `k8s_nodes_force_rejoin`.
- Flannel and `cni0` interfaces are deleted before a reset to avoid the stale bridge error.
- GitLab takes several minutes to come up; the `k8s-gitlab` tag retries `/oauth/token` 40 times.
- `join-command` and `config` are written to the repository root by the master role and read
  by the node role: run both from the same working directory.

## Conventions

- Role structure: `tasks/main.yml`, `defaults/main.yml` for overridable values, `vars/main.yml`
  for pinned versions, `files/` for manifests patched with `sed` or `replace` at run time.
- Versions are pinned in the roles (`k8s_version`, `docker_version`, `linkerd_version`, chart
  versions, Argo CD manifest URL). Bump them there, not in the playbooks.
- Secrets (`vars/scaleway.yml` passwords) are demonstration values; change them before exposing
  the cluster.
- Issue templates for bugs and suggestions are in `.github/ISSUE_TEMPLATE/`; upstream prefers
  suggestions on [flavienbwk/ToDevOps](https://github.com/flavienbwk/ToDevOps).
