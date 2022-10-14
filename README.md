# Injecting Secrets into Terraform Vault

### Contents

#### 1.     Introduction
#### 2.     Pre-requisites
#### 3.     Injecting Secrets into Vault
#### 3.1	    Start Vault Server
#### 3.2	    Cloning the Repo
#### 3.3      Configure AWS Secret Engine into Vault
#### 4.     Provisioning the Resources on AWS
#### 4.1      Destroying the Resources
#### 4.2      Resrtict Vault Role Permissions
#### 4.3      Benefits and considerations
 
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
$ export VAULT_TOKEN=test
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

### 4.0 Provisioning the Resources on AWS

Now that you have successfully configured Vault's AWS Secrets Engine, you can retrieve dynamic short lived AWS token to provision an EC2 instance.

Navigate to the Terraform Operator workspace.

```bash
cd ../operator-workspace
```
In the main.tf file, you should find the following data and resource blocks:

1. the terraform_remote_state.admin data block retrieves the Terraform state file generated from your Vault Admin workspace

2. the vault_aws_access_credentials.creds data block retrieves the dynamic, short-lived AWS credentials from your Vault instance. Notice that this uses the Vault Admin workspace's output variables: backend and role

3. the aws provider is initialized with the short-lived credentials retrieved by vault_aws_access_credentials.creds. The provider is configured to the us-east-1 region, as defined by the region variable

4. the aws_ami.ubuntu data block retrieves the most recent Ubuntu image

5. the aws_instance.main resource block creates an t2.micro EC2 instance

```bash
Tip: We recommend using provider-specific data sources when convenient. terraform_remote_state is more flexible, but requires access to the whole Terraform state.
```

Initialize the Terraform Operator workspace.

```bash
$ terraform init
```

Navigate to the IAM Users page in AWS Console. Search for the username prefix vault-token-terraform-dynamic-aws-creds-vault-admin. Nothing should show up on your initial search. However, a user with this prefix should appear on terraform plan or terraform apply.

Apply the Terraform configuration, remember to confirm the run with a yes. Terraform will provision the EC2 instance using the dynamic credentials generated from Vault.

```bash
$ terraform apply
```

Refresh the IAM Users and search for the vault-token-terraform-dynamic-aws-creds-vault-admin prefix. You should see a IAM user.

![image](https://user-images.githubusercontent.com/50728232/195943367-2c6bc4a0-a363-434f-b95e-54c34df61d0b.png)

This IAM user was generated by Vault with the appropriate IAM policy configured by the Vault Admin workspace. Because the default_lease_ttl_seconds is set to 120 seconds, Vault will revoke those IAM credentials and they will be removed from the AWS IAM console after 120 seconds.

```bash
Tip: The token is generated from the moment the configuration retrieves the temporary AWS credentials (on terraform plan or terraform apply). If the apply run is confirmed after the 120 seconds, the run will fail because the credentials used to initialize the Terraform AWS provider has expired. For these instances or large multi-resource configurations, you may need to adjust the default_lease_ttl_seconds.
```

Navigate to the EC2 page and search for dynamic-aws-creds-operator. You should see an instance provisioned by the Terraform Operator workspace using the short-lived AWS credentials.
![image](https://user-images.githubusercontent.com/50728232/195943566-404084ed-288b-4372-93e2-8d1870227603.png)

Every Terraform run with this configuration will use its own unique set of AWS IAM credentials that are scoped to whatever the Vault Admin has defined.

The Terraform Operator doesn't have to manage long-lived AWS credentials locally. The Vault Admin only has to manage the Vault role rather than numerous, multi-scoped, long-lived AWS credentials.

After 120 seconds, you should see the following in the terminal running Vault.

```bash
2020-07-13T16:07:55.755-0700 [INFO]  expiration: revoked lease: lease_id=dynamic-aws-creds-vault-admin-path/creds/dynamic-aws-creds-vault-admin-role/z1PKR7Y623fk0ZQWW1kwaVVY
```
This shows that Vault has destroyed the short-lived AWS credentials generated for the apply run.

### 4.1 Destroying the Resources

Destroy the EC2 instance, remember to confirm the run with a yes.

```bash
$ terraform destroy
```
This run should have generated and used another set of IAM credentials. Verify that your EC2 instance has been destroyed by viewing the EC2 page of your AWS Console.

### 4.2 Resrtict Vault Role Permissions

If the Vault Admin wanted to remove the Terraform Operator's EC2 permissions, they would only need to update the Vault role's policy.

Navigate to the Vault Admin workspace.

```bash
$ cd ../vault-admin-workspace
```

Remove "ec2:*" from the vault_aws_secret_backend_role.admin resource in your main.tf file.

```bash
$ sed -i '' -e 's/, \"ec2:\*\"//g' main.tf
```

```bash
resource "vault_aws_secret_backend_role" "admin" {
  backend = vault_aws_secret_backend.aws.path
  name    = "${var.name}-role"
  credential_type = "iam_user"

  policy_document = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
-        "iam:*", "ec2:*"
+        "iam:*"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
```

This change restricts the Terraform Operator's ability to provision any AWS EC2 instance.

Apply the Terraform configuration, remember to confirm the run with a yes.

```bash
$ terraform apply
```

### 4.3 Benefits and considerations

This approach to secret injection:

1. alleviates the Vault Admin's responsibility in managing numerous, multi-scoped, long-lived AWS credentials,

2. reduces the risk from a compromised AWS credential in a Terraform run (if a malicious user gains access to an AWS credential used in a Terraform run, that credential is only value for the length of the token's TTL),

3. allows for management of a role's permissions through a Vault role rather than the distribution/management of static AWS credentials,

4. enables development to provision resources without managing local, static AWS credentials

However, this approach may run into issues when applied to large multi-resource configurations. The generated dynamic AWS Credentials are only valid for the length of the token's TTL. As a result, if:

1. the apply process exceeds than the TTL and the configuration needs to provision another resource or

2. the apply confirmation time exceeds the TTL

the apply process will fail because the short-lived AWS Credentials have expired.

You could increase the TTL to conform to your situation; however, this also increases how long the temporary AWS credentials are valid, increasing the malicious actor's attack surface.

