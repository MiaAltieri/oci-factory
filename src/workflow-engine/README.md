# Workflow Engine

The OCI Factory is complemented by a workflow engine which takes care of
actuating upon events of interest. To put it in simple terms, the workflow
engine has the intelligence to decide what happens, and when. On the other
hand, the GitHub workflows in the OCI Factory's repository are simply a set of
loosely coupled jobs for continuously testing and delivering OCI images.

Having said that, the OCI Factory's workflow engine is composed of Temporal Workflows. These Workflows need Temporal Workers and therefore an underlying
infrastructure to host them.

The Temporal server infrastructure already exists, so this directory hosts the
configurations and scripts necessary to deploy:

- [Infrastructure: MicroK8s cluster on OpenStack](#infra)

<a name="infra"></a>

## Infrastructure: MicroK8s cluster on OpenStack

The OpenStack VMs will host the MicroK8s cluster where the Temporal Workers
will be deployed.

### Prerequisites

- `terraform` (>=v1.5.5)
- this TF configuration is designed to use an internal OpenStack
infrastructure, so before proceeding, make sure you:
  - connect to the VPN,
  - forward all the necessary remote endpoints to your `localhost``, with:

      ```bash
      ssh -L 5000:<keystoneUrl>:<keystonePort> -L 9696:<neutronUrl>:<neutronPort> -L 8774:novaApiUrl>:<novaApiPort> <username>@<internalOSServer>
      ```

    this is needed because of Bastion,
- (optional) if you are managing an already existing deployment, **you need**
to request and copy the corresponding `terraform.tfstate` file into this
directory. NOTE: this file may contain sensitive information so please do not
commit or share it.

### Deploy

These Terraform configurations will take care of deploying the VMs and
configuring them with MicroK8s. The `cluster.tf` file has the OpenStack and
MicroK8s configurations, and it relies on `variables.tf` and `cloud-init.yml`.
The former has the secret connection variables that need to be set before
deploying the VMs, while the latter contains the Cloud-init recipes
for configuring said nodes.

 1. set all the required Terraform variables:

    ```bash
    # Find the values in LP
    export TF_VAR_openstack_username=<username>
    export TF_VAR_openstack_password=<password>
    export TF_VAR_openstack_tenant_name=<project/tenant_name>
    export TF_VAR_openstack_region=<region>
    ```

 2. prepare the working directory with `terraform init`
 3. validate the configuration with `terraform validate`
 4. deploy with `terraform apply`
 5. save the resulting `terraform.tfstate` file to allow future management of
 the deployed infrastructure.

When performing any other Terraform command (e.g. `terraform destroy`), the
first 3 steps above are also required if not yet set.

### Access

If successful, you can validate the deployment by accessing the cluster. To do
that, you need the provisioned VMs' IPs:

```bash
terraform show -json | \
    jq '.values.root_module.resources[] |
        select(.type | contains("openstack_compute_instance_v2")) |
        .values.network[].fixed_ip_v4'
```

If you just want the IP of the MicroK8s control plane, then:

```bash
terraform show -json | \
    jq '.values.root_module.resources[] |
        select(.name | contains("rocks-temporal-workers-controller")) |
        .values.network[].fixed_ip_v4'
```

You can then SSH (via `ssh` or `openstack ssh`) into either one of the VMs,
with the `ubuntu` user. The VMs have 2 authorized SSH keys:

 1. a default `rocks-team` key, which is already in place and can be used from
within the ROCKS environment in Scalingstack, and
 2. a key that is dynamically generated by Terraform at deployment time, and
that can be retrieved via

    ```bash
    terraform show -json | \
        jq '.values.root_module.resources[] |
            select(.type | contains("tls_private_key")) | 
            .values.private_key_pem'
    ```

Here's a list of useful operations that can be used to inspect the state of the
cluster, once inside the VMs

- `microk8s status`: to ensure `microk8s` is up and running;
- `kubectl get no`: to ensure the `kubectl` alias is in place and that the
cluster has been properly formed;
- `cat /var/log/cloud-init*`: to inspect the Cloud-init script execution;
- `kubectl get po -A`: to double check that all pods are running, and if not
then `kubectl describe po -A` will dump their information and possibly the
reason why they are failing.

<!-- 
# Deploy OCI Factory Temporal Worker charm for K8s

This is leveraging the existing charm template from
<https://github.com/canonical/temporal-worker-k8s-operator> and adding
additional workflows and activities for the work we need done within the
context of the OCI Factory.

## Prerequisites

The Microk8s infrastructure must already be in place. -->
