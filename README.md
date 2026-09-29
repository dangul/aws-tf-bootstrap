# AWS Terraform Bootstrap

Terraform configuration for creating foundational AWS resources for Terraform state:

- An S3 bucket with versioning, AES-256 encryption, and public access blocked.
- A DynamoDB table for state locking.

The default AWS region is `eu-north-1`. You must provide a globally unique bucket name. The default DynamoDB table name is `terraform-locks`.

## Prerequisites

- Terraform `>= 1.5`.
- The AWS CLI and permission to create S3 buckets and DynamoDB tables in your AWS account.
- AWS credentials configured for Terraform, for example through an AWS CLI profile or environment variables. Never put access keys in Terraform files or commit them to Git.

Verify which AWS account you are using before proceeding:

```sh
aws sts get-caller-identity
```

For a named AWS CLI profile, you can sign in and select it like this:

```sh
aws sso login --profile my-profile
export AWS_PROFILE=my-profile
```

## Initialize and create resources

Run these commands from the repository root. Replace the bucket name with your own globally unique name.

```sh
terraform init
terraform validate
terraform plan -var="state_bucket_name=my-unique-terraform-state-name"
terraform apply -var="state_bucket_name=my-unique-terraform-state-name"
```

Review the plan before approving `apply`. Display the names of the created resources with:

```sh
terraform output
```

To change the AWS region:

```sh
terraform plan \
  -var="aws_region=eu-west-1" \
  -var="state_bucket_name=my-unique-terraform-state-name"
```

Use the same variable values for subsequent `plan`, `apply`, and `destroy` commands. Local `.tfvars` files are excluded from Git by `.gitignore`.

## Terraform state

This configuration creates resources that can be used for remote state storage and locking, but it does not configure an S3 backend by itself. Until you add a backend, Terraform stores state locally in `terraform.tfstate`. Do not publish or share this state file; it may contain sensitive values.

After creating the resources, you can move state to S3 by adding the following to the `terraform` block in `main.tf`. Replace the bucket name with the value you used above:

```hcl
backend "s3" {
  bucket         = "my-unique-terraform-state-name"
  key            = "bootstrap/terraform.tfstate"
  region         = "eu-north-1"
  dynamodb_table = "terraform-locks"
  encrypt        = true
}
```

Then run `terraform init -migrate-state` and follow Terraform's prompt to migrate the local state file. Backend settings cannot use regular Terraform variables, so the bucket name must be specified directly in the block.

## Destroy resources

`terraform destroy` removes the resources because `prevent_destroy` is not enabled in this configuration. If you enabled the S3 backend, first remove the backend block from `main.tf` and run `terraform init -migrate-state` to move state back to local storage. The bucket's state object must be removed before Terraform can delete the bucket. Then use the same variables as when creating the resources:

```sh
terraform destroy -var="state_bucket_name=my-unique-terraform-state-name"
```