# Jenkins X EKS Module
<a id="markdown-Jenkins%20X%20EKS%20Module" name="Jenkins%20X%20EKS%20Module"></a>

![Terraform Version](https://img.shields.io/badge/tf-%3E%3D0.12.0-blue.svg)

This repository contains a Terraform module for creating an EKS cluster and all the necessary infrastructure to install Jenkins X via `jx boot`.
The module generates for this purpose a  templated `jx-requirements.yml` file for `jx boot`.

The module makes use of the [Terraform EKS cluster Module](https://github.com/terraform-aws-modules/terraform-aws-eks).

<!-- TOC depthfrom:2 -->

- [What is a Terraform module](#what-is-a-terraform-module)
- [How do you use this module](#how-do-you-use-this-module)
    - [Prerequisites](#prerequisites)
    - [Cluster provisioning](#cluster-provisioning)
        - [Inputs](#inputs)
        - [Outputs](#outputs)
    - [Long Term Storage](#long-term-storage)
    - [Vault](#vault)
    - [ExternalDNS](#externaldns)
    - [cert-manager](#cert-manager)
    - [Running `jx boot`](#running-jx-boot)
    - [Production cluster considerations](#production-cluster-considerations)
    - [Configuring a Terraform backend](#configuring-a-terraform-backend)
    - [Examples](#examples)
- [FAQ: Frequently Asked Questions](#faq-frequently-asked-questions)
    - [IAM Roles for Service Accounts](#iam-roles-for-service-accounts)
- [Development](#development)
    - [Releasing](#releasing)
- [How can I contribute](#how-can-i-contribute)

<!-- /TOC -->

## What is a Terraform module
<a id="markdown-What%20is%20a%20Terraform%20module" name="What%20is%20a%20Terraform%20module"></a>

A Terraform module refers to a self-contained package of Terraform configurations that are managed as a group.
For more information about modules refer to the Terraform [documentation](https://www.terraform.io/docs/modules/index.html).

## How do you use this module
<a id="markdown-How%20do%20you%20use%20this%20module" name="How%20do%20you%20use%20this%20module"></a>

### Prerequisites
<a id="markdown-Prerequisites" name="Prerequisites"></a>

This Terraform module allows you to create an [EKS](https://aws.amazon.com/eks/) cluster for installation of Jenkins X.
You need the following binaries locally installed and configured on your _PATH_:

- `terraform` (~> 0.12.0)
- `kubectl` (>=1.10)
- `aws-iam-authenticator`

### Cluster provisioning
<a id="markdown-Cluster%20provisioning" name="Cluster%20provisioning"></a>

A default Jenkins X ready cluster can be provisioned by creating a _main.tf_ file in an empty directory with the following content:

```terraform
module "eks-jx" {
  source  = "jenkins-x/eks-jx/aws"

  output "vault_user_id" {
    value       = module.eks-jx.vault_user_id
    description = "The Vault IAM user id"
  }

  output "vault_user_secret" {
    value       = module.eks-jx.vault_user_secret
    description = "The Vault IAM user secret"
  }
}
```

Due to the Vault issue [7450](https://github.com/hashicorp/vault/issues/7450), this Terraform module needs for now to create a new IAM user for installing Vault.
It also creates an IAM access key whose id and secret are defined in the output above.
You need the id and secret for running [`jx boot`](#running-jx-boot).

If you do not want Terraform to create a new IAM user or you do not have permissions to create one, you need to provide the name of an existing IAM user.

```terraform
module "eks-jx" {
  source  = "jenkins-x/eks-jx/aws"

  vault_user="<your_vault_iam_username>"
}
```

The IAM user does not need any permissions attached to it.
For more information, refer to [Configuring Vault for EKS](https://jenkins-x.io/docs/getting-started/setup/boot/clouds/amazon/#configuring-vault-for-eks) in the Jenkins X documentation.

Once you have your initial configuration, you can apply it by running:

```sh
terraform init
terraform apply
```

This creates an EKS cluster with all possible configuration options defaulted.

You then need to export the environment variables _VAULT_AWS_ACCESS_KEY_ID_ and _VAULT_AWS_SECRET_ACCESS_KEY_.

```sh
export VAULT_AWS_ACCESS_KEY_ID=$(terraform output vault_user_id)
export VAULT_AWS_SECRET_ACCESS_KEY=$(terraform output vault_user_secret)
```

If you specified _vault_user_ you need to provide the access key id and secret for the specified user.

:warning: **Note**: This example is for getting up and running quickly.
It is not intended for a production cluster.
Refer to [Production cluster considerations](#production-cluster-considerations) for things to consider when creating a production cluster.

The following sections provide a full list of configuration in- and output variables.

#### Inputs
<a id="markdown-Inputs" name="Inputs"></a>

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:-----:|
| apex\_domain | The main domain to either use directly or to configure a subdomain from | `string` | `""` | no |
| cluster\_name | Variable to provide your desired name for the cluster. The script will create a random name if this is empty | `string` | `""` | no |
| create\_and\_configure\_subdomain | Flag to create an NS record set for the subdomain in the apex domain's Hosted Zone | `bool` | `false` | no |
| desired\_node\_count | The number of worker nodes to use for the cluster | `number` | `3` | no |
| enable\_external\_dns | Flag to enable or disable External DNS in the final `jx-requirements.yml` file | `bool` | `false` | no |
| enable\_logs\_storage | Flag to enable or disable long term storage for logs | `bool` | `true` | no |
| enable\_reports\_storage | Flag to enable or disable long term storage for reports | `bool` | `true` | no |
| enable\_repository\_storage | Flag to enable or disable the repository bucket storage | `bool` | `true` | no |
| enable\_tls | Flag to enable TLS in the final `jx-requirements.yml` file | `bool` | `false` | no |
| max\_node\_count | The maximum number of worker nodes to use for the cluster | `number` | `5` | no |
| min\_node\_count | The minimum number of worker nodes to use for the cluster | `number` | `3` | no |
| node\_machine\_type | The instance type to use for the cluster's worker nodes | `string` | `"m5.large"` | no |
| production\_letsencrypt | Flag to use the production environment of letsencrypt in the `jx-requirements.yml` file | `bool` | `false` | no |
| region | The region to create the resources into | `string` | `"us-east-1"` | no |
| subdomain | The subdomain to be added to the apex domain. If subdomain is set, it will be appended to the apex domain in  `jx-requirements-eks.yml` file | `string` | `""` | no |
| tls\_email | The email to register the LetsEncrypt certificate with. Added to the `jx-requirements.yml` file | `string` | `""` | no |
| vault\_user | The AWS IAM Username whose credentials will be used to authenticate the Vault pods against AWS | `string` | `""` | no |
| vpc\_cidr\_block | The vpc CIDR block | `string` | `"10.0.0.0/16"` | no |
| vpc\_name | The name of the VPC to be created for the cluster | `string` | `"tf-vpc-eks"` | no |
| vpc\_subnets | The subnet CIDR block to use in the created VPC | `list(string)` | <pre>[<br>  "10.0.1.0/24",<br>  "10.0.2.0/24",<br>  "10.0.3.0/24"<br>]</pre> | no |

#### Outputs
<a id="markdown-Outputs" name="Outputs"></a>

| Name | Description |
|------|-------------|
| cert\_manager\_iam\_role | The IAM Role that the Cert Manager pod will assume to authenticate |
| cluster\_name | The name of the created cluster |
| cm\_cainjector\_iam\_role | The IAM Role that the CM CA Injector pod will assume to authenticate |
| controllerbuild\_iam\_role | The IAM Role that the ControllerBuild pod will assume to authenticate |
| external\_dns\_iam\_role | The IAM Role that the External DNS pod will assume to authenticate |
| jxui\_iam\_role | The IAM Role that the Jenkins X UI pod will assume to authenticate |
| lts\_logs\_bucket | The bucket where logs from builds will be stored |
| lts\_reports\_bucket | The bucket where test reports will be stored |
| lts\_repository\_bucket | The bucket that will serve as artifacts repository |
| tekton\_bot\_iam\_role | The IAM Role that the build pods will assume to authenticate |
| vault\_dynamodb\_table | The Vault DynamoDB table |
| vault\_kms\_unseal | The Vault KMS Key for encryption |
| vault\_unseal\_bucket | The Vault storage bucket |
| vault\_user\_id | The Vault IAM user id |
| vault\_user\_secret | The Vault IAM user secret |

### Long Term Storage
<a id="markdown-Long%20Term%20Storage" name="Long%20Term%20Storage"></a>

You can choose to create S3 buckets for long term storage and enable them in the generated _jx-requirements.yml_ file with `enable_logs_storage`, `enable_reports_storage` and `enable_repository_storage`.

During `terraform apply` the enabledS3 buckets are created, and the generated `jx-requirements.yml` will contain the following section:

```yaml
    storage:
      logs:
        enabled: ${enable_logs_storage}
        url: s3://${logs_storage_bucket}
      reports:
        enabled: ${enable_reports_storage}
        url: s3://${reports_storage_bucket}
      repository:
        enabled: ${enable_repository_storage}
        url: s3://${repository_storage_bucket}
```

### Vault
<a id="markdown-Vault" name="Vault"></a>

Vault is used by Jenkins X for managing secrets.
Part of this module's responsibilities is the creation of all resources required to run the [Vault Operator](https://github.com/banzaicloud/bank-vaults).
These resources are An S3 Bucket, a DynamoDB Table and a KMS Key.

### ExternalDNS
<a id="markdown-ExternalDNS" name="ExternalDNS"></a>

You can enable [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) with the `enable_external_dns` variable. This modifies the generated _jx-requirements.yml_ file to enable External DNS when running `jx boot`.

If `enable_external_dns` is _true_, additional configuration is required.

If you want to use a domain with an already existing Route 53 Hosted Zone, you can provide it through the `apex_domain` variable:

This domain will be configured in the resulting _jx-requirements.yml_ file in the following section:

```yaml
    ingress:
      domain: ${domain}
      ignoreLoadBalancer: true
      externalDNS: ${enable_external_dns}
```

If you want to use a subdomain and have this module create and configure a new Hosted Zone with DNS delegation, you can provide the following variables:

`subdomain`: This subdomain is added to the apex domain and configured in the resulting _jx-requirements.yml_ file.

`create_and_configure_subdomain`: This flag instructs the script to create a new `Route53 Hosted Zone` for your subdomain and configure DNS delegation with the apex domain.

By providing these variables, the script creates a new `Route 53` HostedZone that looks like `<subdomain>.<apex_domain>`, then it delegates the resolving of DNS to the apex domain.
This is done by creating a `NS` RecordSet in the apex domain's Hosted Zone with the subdomain's HostedZone nameservers.

This ensures that the newly created HostedZone for the subdomain is instantly resolvable instead of having to wait for DNS propagation.

### cert-manager
<a id="markdown-cert-manager" name="cert-manager"></a>

You can enable [cert-manager](https://github.com/jetstack/cert-manager) to use TLS for your cluster through LetsEncrypt with the `enable_tls` variable.

[LetsEncrypt](https://letsencrypt.org/) has two environments, `staging` and `production`.

If you use staging, you will receive self-signed certificates, but you are not rate-limited, if you use the `production` environment, you receive certificates signed by LetsEncrypt, but you can be rate limited.

You can choose to use the `production` environment with the `production_letsencrypt` variable:

You need to provide a valid email to register your domain in LetsEncrypt with `tls_email`:

### Running `jx boot`
<a id="markdown-Running%20%60jx%20boot%60" name="Running%20%60jx%20boot%60"></a>

An output of applying this Terraform module is a _jx-requirements.yml_ file in the current directory.
This file can be used as input to [Jenkins X Boot](https://jenkins-x.io/docs/getting-started/setup/boot/) which is responsible for installing all the required Jenkins X components into the cluster created by this module.

Change into  an empty directory and execute:

```bash
jx boot --requirements <path-to-jx-requirements.yml>
```

:grey_exclamation: Remember you need to export _VAULT_AWS_ACCESS_KEY_ID_ and _VAULT_AWS_SECRET_ACCESS_KEY_.
See [Cluster provisioning](#cluster-provisioning).

You are prompted for any further required configuration.
The number of prompts depends on how much you have [pre-configured](#inputs) via your Terraform variables.

### Production cluster considerations
<a id="markdown-Production%20cluster%20considerations" name="Production%20cluster%20considerations"></a>

The configuration, as seen in [Cluster provisioning](#cluster-provisioning), is not suited for creating and maintaining a production Jenkins X cluster.
The following is a list of considerations for a production use case.

- Specify the version attribute of the module, for example:

    ```terraform
    module "jx" {
      source  = "jenkins-x/eks-jx/aws"
      version = "1.0.0"
      # insert your configuration
    }
    ```

  Specifying the version ensures that you are using a fixed version and that version upgrades cannot occur unintended.

- Keep the Terraform configuration under version control by creating a dedicated repository for your cluster configuration or by adding it to an already existing infrastructure repository.

- Setup a Terraform backend to securely store and share the state of your cluster. For more information refer to [Configuring a Terraform backend](#configuring-a-terraform-backend).

### Configuring a Terraform backend
<a id="markdown-Configuring%20a%20Terraform%20backend" name="Configuring%20a%20Terraform%20backend"></a>

A "[backend](https://www.terraform.io/docs/backends/index.html)" in Terraform determines how state is loaded and how an operation such as _apply_ is executed.
By default, Terraform uses the _local_ backend, which keeps the state of the created resources on the local file system.
This is problematic since sensitive information will be stored on disk and it is not possible to share state across a team.
When working with AWS a good choice for your Terraform backend is the [_s3_ backend](https://www.terraform.io/docs/backends/types/s3.html)  which stores the Terraform state in an AWS S3 bucket.
The [examples](./examples) directory of this repository contains configuration examples for using the _s3_ backed.

To use the _s3_ backend, you will need to create the bucket upfront.
You need the S3 bucket as well as a Dynamo table for state locks.
You can use [terraform-aws-tfstate-backend](https://github.com/cloudposse/terraform-aws-tfstate-backend) to create these required resources.

### Examples
<a id="markdown-Examples" name="Examples"></a>

You can find examples for different configurations in the [examples folder](./examples).

Each example generates a valid _jx-requirements.yml_ file that can be used to boot a Jenkins X cluster.

## FAQ: Frequently Asked Questions
<a id="markdown-FAQ%3A%20Frequently%20Asked%20Questions" name="FAQ%3A%20Frequently%20Asked%20Questions"></a>

### IAM Roles for Service Accounts
<a id="markdown-IAM%20Roles%20for%20Service%20Accounts" name="IAM%20Roles%20for%20Service%20Accounts"></a>

This module sets up a series of IAM Policies and Roles. These roles will be annotated into a few Kubernetes Service accounts.
This allows us to make use of [IAM Roles for Sercive Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) to set fine-grained permissions on a pod per pod basis.
There is no way to provide your own roles or define other Service Accounts by variables, but you can always modify the `eks/terraform/jx/irsa.tf` Terraform file.

## Development
<a id="markdown-Development" name="Development"></a>

### Releasing
<a id="markdown-Releasing" name="Releasing"></a>

At the moment, there is no release pipeline defined in [jenkins-x.yml](./jenkins-x.yml).
A Terraform release does not require building an artifact; only a tag needs to be created and pushed.
To make this task easier and there is a helper script `release.sh` which simplifies this process and creates the changelog as well:

```sh
./scripts/release.sh
```

This can be executed on demand whenever a release is required.
For the script to work, the environment variable _$GH_TOKEN_ must be exported and reference a valid GitHub API token.

## How can I contribute
<a id="markdown-How%20can%20I%20contribute" name="How%20can%20I%20contribute"></a>

Contributions are very welcome! Check out the [Contribution Guidelines](./CONTRIBUTING.md) for instructions.
