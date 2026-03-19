# Jenkins + Terraform Pipeline (AWS EC2)

This project demonstrates an end-to-end CI workflow where **Jenkins runs Terraform** to provision (or destroy) an **AWS EC2 instance**.
The repository contains:
- A `Jenkinsfile` that performs: checkout -> `terraform init` -> `terraform plan` -> (manual/auto) approval -> `terraform apply` or `terraform destroy`
- Terraform code (`*.tf`) that creates an EC2 instance and outputs its `public_ip` and `instance_id`
Repo: https://github.com/FaizReza/Jenkins-Terraform-Pipeline

---
## Architecture (High Level)
1. Jenkins starts a pipeline job (parameterized).
2. Jenkins runs Terraform:
   - Initializes providers/plugins
   - Generates an execution plan and saves it as `tfplan.txt`
3. Jenkins either:
   - Requires manual approval before `apply` (unless `autoApprove=true`)
   - Or immediately destroys the infrastructure for `destroy`
---
## Prerequisites
### Jenkins requirements
- Jenkins installed with Pipeline support
- Terraform installed on the Jenkins agent (must be available as `terraform` in `PATH`)
- AWS credentials configured in Jenkins
### Jenkins Credentials (required by `Jenkinsfile`)
The pipeline uses these Jenkins credential IDs:
- `aws-access-key-id`
- `aws-secret-access-key`
Make sure IAM permissions allow at least:
- EC2: `RunInstances`, `TerminateInstances`, `DescribeInstances`
- (and any required read permissions used by Terraform)
---
## How to Run (Jenkins)
1. Create a Jenkins **Pipeline job**
2. Point it to this repo and use the `Jenkinsfile`
3. Trigger the pipeline with parameters:
Parameters in `Jenkinsfile`:
- `action`: choose one of
  - `apply` (create/launch EC2)
  - `destroy` (terminate EC2)
- `autoApprove`:
  - `false`: pipeline pauses for manual approval (uses `tfplan.txt`)
  - `true`: pipeline applies immediately (for `apply` only)
Example expected flow:
- `action=apply`, `autoApprove=false`:
  - pipeline runs plan, then asks for approval
  - if approved -> `terraform apply ... tfplan`
---
## What Terraform Creates
Terraform resource (in `main.tf`):
- `aws_instance.public_instance`
Outputs (in `output.tf`):
- `public_ip` (EC2 public IP)
- `instance_id` (EC2 instance ID)
Note: `main.tf` also defines a data source `aws_ami "myami"`, but the EC2 instance currently uses a hardcoded AMI value.
---
## Terraform Configuration Notes (Important)
### 1) Region mismatch
- `Jenkinsfile` sets: `AWS_DEFAULT_REGION = 'us-east-1'`
- `provider.tf` uses `var.aws_region` with a default of `eu-north-1`
In the current pipeline, Terraform is executed without `-var` flags, so Terraform will typically use the default from `variables.tf`.
If you want the region to be `us-east-1`, you can:
- update `variables.tf` default `aws_region`, or
- set `TF_VAR_aws_region` in the Jenkins job/environment, or
- modify the pipeline to pass `-var aws_region=...`
### 2) Checkout URL inside Jenkinsfile
In `Jenkinsfile`, the checkout step points to:
- `https://github.com/FaizReza/Jenkins-Terraform-Pipeline.git`
If you want Jenkins to use *this* repo, update that URL to your GitHub repo URL.
### 3) AMI handling
`main.tf` has:
- a data AMI lookup (`aws_ami.myami`)
- but the actual instance uses a hardcoded AMI value
Recommended improvement (optional): switch `aws_instance.public_instance.ami` to use `var.ami` or the data source output.
---
## Common Troubleshooting
- **Credentials error**: verify Jenkins credentials IDs exactly match:
  - `aws-access-key-id`
  - `aws-secret-access-key`
- **AMI not found**: AMI IDs are region-specific. Confirm the AMI used matches the Terraform region.
- **Terraform not found**: install Terraform on the Jenkins agent and ensure it’s in `PATH`.
---
## Suggested Enhancements (Optional)
- Add a final stage to run `terraform output` after `apply` to print `public_ip` and `instance_id`
- Add `-var` / `TF_VAR_` wiring for `aws_region`, `instance_type`, and `name_tag`
- Fix the checkout URL to point to `FaizReza/Jenkins-Terraform-Pipeline`
- Use the discovered AMI (`data.aws_ami.myami`) instead of a hardcoded AMI
