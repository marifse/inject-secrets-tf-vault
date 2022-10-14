# Injecting Secrets into Terraform Vault

### Contents

#### 1.     Introduction
#### 2.     Pre-requisites
#### 3.     Injecting Secrets into Vault
#### 3.1	    Start Vault Server
#### 3.2	    Cloning the Repo
#### 3.3      Configure AWS Secret Engine into Vault
#### 4.     Provisioning the Resources on AWS

 
### 1.0 Introduction

Usually developers and DevOps engineers are required to give the credentials for deploying application on cloud accounts, which could be long live, and thus give them the freedom to exploit and make unauthorized use of them. This can be dangerous for the owners.

To avoid this, we can use the Hashicorp Vault Secret Engine to generate appropriately scoped and short lived credentails to be used by the developers.

In this tutorial, you will learn how to start the vault and push/inject the secrets into terraform vault and provision the resources using those short lived credentials.

### 2.0 Pre-requisites

To use this tutorial, you should be familiar with terraform basic commands, and vault basic commands. And should have the following running:

  -	Terraform installed locally - [How to Install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
  -	Vault installed locally - [How to install Terraform Vault](https://learn.hashicorp.com/tutorials/vault/getting-started-install)
  -	An AWS account and AWS_Access_key/AWS_Secret_Key credentials - [Create Credentails for AWS User](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/getting-your-credentials.html)

### 3.0	Injecting Secrets into Vault

To make the credentials short termed and make them provide to developers, you would have to provide the terraform vault with the AWS credentails and then the Terraform Vault can create the short termed credentials and being them used by developers.

To inject the secrets into vault, we will first start the Vault server, clone the repo which contains the Vault configuration, and the AWS resources provisioned script  by using Vault short term credentials.


### 3.1 Start the Vault Server

Start the Vault server with the below command in development mode with root token as test. 

```bash
$ vault server -dev -dev-root-token-id="test"
```
Your Vault server should be running up now, browse to localhost:8200 or 127.0.0.1:8200 and login into the instance using your root token: test

![image](https://user-images.githubusercontent.com/50728232/195939638-2da00fd2-3205-4a36-a0aa-6af54f058da4.png)


### 3.2 Cloning the Repo

Clone the [inject-secrets-tf-vault](https://github.com/hashicorp/learn-terraform-inject-secrets-aws-vault) with the below command, this repo contains the configuration to inject the secrets into the vault.

```bash
$ git clone https://github.com/hashicorp/learn-terraform-inject-secrets-aws-vault && cd learn-terraform-inject-secrets-aws-vault
```
The directory contains two sub-directories: Vault Admin Workspace & Operator Workspace, one for terraform injecting secrets configuration and the other for provisioning the resources on AWS.


### 3.3	Configure AWS Secret Engine into Vault

In another terminal window (leave the Vault instance running), navigate to the Vault Admin directory.

```bash
$ cd vault-admin-workspace
```

In the main.tf file, you will find 2 resources:

the vault_aws_secret_backend.aws resource configures AWS Secrets Engine to generate a dynamic token that lasts for 2 minutes.

the vault_aws_secret_backend_role.admin resource configures a role for the AWS Secrets Engine named dynamic-aws-creds-vault-admin-role with an IAM policy that allows it iam:* and ec2:* permissions.

This role will be used by the Terraform Operator workspace to dynamically generate AWS credentials scoped to this IAM policy.

Before applying this configuration, set the required Terraform variable substituting <AWS_ACCESS_KEY_ID> and <AWS_SECRET_ACCESS_KEY> with your AWS Credentials. Notice that we're also setting the required Vault Provider arguments as environment variables: VAULT_ADDR & VAULT_TOKEN.

```bash
$ export TF_VAR_aws_access_key=<AWS_ACCESS_KEY_ID>
$ export TF_VAR_aws_secret_key=<AWS_SECRET_ACCESS_KEY>
$ export VAULT_ADDR=http://127.0.0.1:8200
$ export VAULT_TOKEN=education
```
Initialize the Vault Admin workspace.
```bash
$ terraform init
```

In your initialized directory, run terraform apply, review the planned actions, and confirm the run with a yes

```bash
$ terraform apply

## ...

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

backend = "dynamic-aws-creds-vault-admin-path"
role = "dynamic-aws-creds-vault-admin-role"
```
Notice that there are two output variables named backend and role. These output variables will be used by the Terraform Operator workspace in a later step.

If you go to the terminal where your Vault server is running, you should see Vault output something similar to the below. This means Terraform was successfully able to mount the AWS Secrets Engine at the specified path. The role has also been configured although it's not output in the logs.

```bash
[INFO]  core: successful mount: namespace= path=dynamic-aws-creds-vault-admin-path/ type=aws
```

### 3.3 To a new Kubernetes cluster

Clone the repo as told in above step 3.0, and follow the below steps. 

• Go into sub-directory [(with-new-cluster)](https://github.com/marifse/nodejs-cloudant/tree/master/terraform/simple-kube/classic-pipeline/new-infra) of cloned repo with below command.

```bash
cd nodejs-cloudant/terraform/simple-kube/classic-pipeline/new-infra/
```

• Replace the **API key** value with your key and set the **Kubernetes cluster name, Cloudant database name, and the container registry namespace** and other variables as desired.

•	Initialize the repo with below command.

```bash
terraform init
```

•	Deploy NodeJS with below terraform command.

```bash
terraform apply
```

• Confirm with **yes**.

Once all the resources have been provisioned, you can go to the Toolchain service in IBM Cloud console and in deployed region, you would find the delivery pipeline, there would be three stages, and in third deployment stage, you would find the URL for your NodeJS application deployed over there. You can open that URL in browser and see your application running.

•	To destroy the deployment run below terraform command.

```bash
terraform destroy
```

### 3.2.2 To an existing Kubernetes cluster

Clone the repo as mentioned in above step 3.0, and follow the steps below. 

• Go into sub-directory [with-existing-cluster](https://github.com/marifse/nodejs-cloudant/tree/master/terraform/simple-kube/classic-pipeline/on-existing-cluster-cloudant) of cloned repo with below command.

```bash
cd nodejs-cloudant/terraform/simple-kube/classic-pipeline/on-existing-cluster-cloudant
```

•	Replace the **API key** value with your key and set the **Kubernetes cluster name** and the **Cloudant database name** with your existing IKS cluster name and Cloudant DB name, and other variables as desired.

•	Initialize the repo with below terraform command.

```bash
terraform init
```

•	Deploy NodeJS with below terraform command.

```bash
terraform apply
```

• Confirm with **yes**.

Once all the resources have been provisioned, you can go to the Toolchain service in IBM Cloud console, and in deployed region you would find the delivery pipeline, there would be three stages, and in third deployment stage, you would find the URL for your NodeJS application deployed over there. You can open that URL in browser and see your application running.

•	To destroy the deployment run below terraform command.

```bash
terraform destroy
```

