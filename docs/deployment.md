# Deployment

This is the complete walkthrough, step by step, kept from the upstream README. It runs from
your computer with Terraform and Ansible installed. Every Ansible role is idempotent: run it
again whenever a step fails and the role continues from the current state.

## 1. Instantiate the infrastructure

[Scaleway](https://www.scaleway.com/en/) is the cloud provider. Three VMs with 8 GB of RAM are
recommended; `plans/scaleway.tf` creates DEV1-L instances.

1. In your Scaleway account, open [Credentials](https://console.scaleway.com/project/credentials)
   and create an API key named `ToDevOps`.
2. Export the values of your project and key:

    ```bash
    export TF_VAR_SCW_PROJECT_ID="my-project-id"
    export TF_VAR_SCW_ACCESS_KEY="my-access-key"
    export TF_VAR_SCW_SECRET_KEY="my-secret-key"
    ```

3. Check the plan:

    ```bash
    cd ./plans
    terraform init
    terraform plan
    ```

4. Apply it:

    ```bash
    terraform apply
    ```

5. Write the Terraform outputs into the Ansible inventory:

    ```bash
    sudo apt install -y jq               # JSON parser used by the script
    terraform output -json > terraform_values.json
    cd ..
    bash terraform_to_ansible_values.sh  # replaces the IPs and machine names in inventories/scaleway.ini
    ```

## 2. Deploy the infrastructure services

This step builds the Kubernetes cluster, its services and GitLab.

1. Check the values in `./inventories/scaleway.ini` (three hosts, `root` user).
2. Adjust `./vars/scaleway.yml` to your infrastructure if needed (the defaults work).
3. Register the local domain names in your `/etc/hosts`:

    ```bash
    ansible-playbook -i inventories/scaleway.ini ./set_local_hosts.yml --extra-vars @./vars/scaleway.yml --ask-become-pass
    ```

4. Run the installation, one tag at a time:

    ```bash
    # DNS entries, Docker, kubectl and Helm on the three VMs
    ansible-playbook -i inventories/scaleway.ini ./playbooks/infrastructure.yml -t base --extra-vars @./vars/scaleway.yml

    # Kubernetes control plane, nodes, ingress, MetalLB, OpenEBS, dashboard, HAProxy
    ansible-playbook -i inventories/scaleway.ini ./playbooks/infrastructure.yml -t k8s-setup --extra-vars @./vars/scaleway.yml

    # GitLab on Kubernetes
    # https://gitlab.todevops.local/ (username: root, default password: mySuperSecurePassword)
    ansible-playbook -i inventories/scaleway.ini ./playbooks/infrastructure.yml -t k8s-gitlab --extra-vars @./vars/scaleway.yml

    # Linkerd on Kubernetes
    # http://linkerd.todevops.local/ (username: admin, default password: admin)
    ansible-playbook -i inventories/scaleway.ini ./playbooks/infrastructure.yml -t k8s-linkerd --extra-vars @./vars/scaleway.yml

    # Fluentd, Elasticsearch and Kibana on Kubernetes
    # http://kibana.todevops.local/ (no authentication)
    ansible-playbook -i inventories/scaleway.ini ./playbooks/infrastructure.yml -t k8s-logging --extra-vars @./vars/scaleway.yml

    # Argo CD on Kubernetes
    # https://argocd.todevops.local/ (username: admin, default password: mySuperSecurePassword)
    ansible-playbook -i inventories/scaleway.ini ./playbooks/infrastructure.yml -t k8s-argocd --extra-vars @./vars/scaleway.yml
    ```

    `k8s-gitlab` waits up to ten minutes for GitLab to answer on `/oauth/token` before creating
    the admin users listed in `gitlab_admins`.

5. (optional) Remove the convenience ingresses

    Linkerd, Kibana and the Argo CD UI are exposed through ingresses for convenience. This is
    NOT recommended beyond a test cluster: use `kubectl port-forward` to reach the services
    instead. For example, `kubectl port-forward -n linkerd-viz svc/web 8080:8084` on the
    master, then `ssh -L 8080:localhost:8080 root@<VM-IP>` from your computer.

    ```bash
    kubectl delete ingress -n linkerd-viz web-ingress
    kubectl delete ingress -n kube-logging kibana-ingress
    kubectl delete ingress -n argocd argocd-ingress
    ```

## 3. Set up continuous deployment

The infrastructure is ready. Argo CD now needs a repository to watch: the playbook imports
`public_repository_to_clone` (by default
[Nginx-Template-Deployment](https://github.com/verovec/Nginx-Template-Deployment); upstream
used the [reactjs-flask-ldap-boilerplate](https://github.com/flavienbwk/reactjs-flask-ldap-boilerplate))
into the GitLab instance and registers it in Argo CD.

```bash
# Registers the master's SSH key on the root GitLab account and pushes the repository
ansible-playbook -i inventories/scaleway.ini ./playbooks/setup-continuous-deployment.yml -t import-repository --extra-vars @./vars/scaleway.yml --extra-vars "gitlab_password={{ gitlab_root_password }}"

# Configures Argo CD for that repository
ansible-playbook -i inventories/scaleway.ini ./playbooks/setup-continuous-deployment.yml -t configure-argocd-repository --extra-vars @./vars/scaleway.yml --extra-vars "gitlab_user=root"
```

The Argo CD `Application` syncs the `manifests/` folder of the repository into the `my-app`
namespace with `selfHeal` and `prune`: a push to GitLab is deployed, a manual change in the
cluster is reverted.

## Where things are after the run

| What | Where |
|------|-------|
| `kubectl` access | `root@master`, `~/.kube/config`; a copy is fetched to `./config` (ignored by git) |
| Join command of the nodes | `./join-command` (ignored by git) |
| Wildcard certificate | `/root/todevops.local.{crt,key,pem}` on the master |
| Registry credentials | `regcred` secret in the `default` namespace |
| Argo CD manifests | `/tmp/argocd/` on the master |
