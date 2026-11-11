# DAY 11 — Terraform Import, State MV and State RM

## 1. The common idea

Terraform state maps Terraform resource addresses to real infrastructure objects. These commands change that mapping; they do not all perform the same operation.

| Command | State effect | Directly deletes real infrastructure? |
|---|---|---|
| terraform import ADDRESS ID | Adds an existing object to a Terraform address | No |
| terraform state mv SOURCE DESTINATION | Moves a state binding to a new address | No |
| terraform state rm ADDRESS | Removes a binding from state | No |

- **Import:** Adopt an existing object that is not tracked by this state.
- **State mv:** Preserve the same object while changing its Terraform address.
- **State rm:** Stop tracking an object without deleting it.

Always back up state, coordinate with other operators, inspect exact addresses, and review a fresh plan before applying.

## 2. Small configuration

Use a disposable AWS account and an S3 bucket that already exists. Replace the example name with a globally unique bucket name.

~~~hcl
terraform {
  required_version = ">= 1.10.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "app" {
  bucket = "company-app-dev-unique-12345"

  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
~~~

Initialize and inspect:

~~~bash
terraform init
terraform fmt -check
terraform validate
terraform plan
terraform state list
~~~

Before import, the state list should not contain aws_s3_bucket.app.

## 3. terraform import — adopt an existing object

### Scenario

An operator created the S3 bucket manually in AWS. Terraform configuration now declares that bucket, but the current state does not know about it.

~~~bash
terraform import \
  aws_s3_bucket.app \
  company-app-dev-unique-12345
~~~

Arguments:
1. Terraform address: aws_s3_bucket.app
2. Provider-specific import ID: company-app-dev-unique-12345

Inspect the result:

~~~bash
terraform state list
terraform state show aws_s3_bucket.app
terraform plan
~~~

Expected address:

~~~text
aws_s3_bucket.app
~~~

Important:
- Import does not create the AWS bucket.
- Import does not generate the complete Terraform configuration.
- The resource block must exist in configuration for the CLI import workflow.
- Import IDs are provider/resource-specific; they are not always an ARN or a name.
- After import, reconcile configuration with the actual object. Do not apply unexplained changes.

### Declarative import blocks

Terraform also supports an import block:

~~~hcl
import {
  to = aws_s3_bucket.app
  id = "company-app-dev-unique-12345"
}
~~~

Then run:

~~~bash
terraform plan
terraform apply
~~~

The import block makes adoption visible in code review and lets Terraform plan the import. The CLI import command performs the state operation directly. Choose one workflow; do not run both against the same address unnecessarily.

## 4. terraform state mv — change the address, preserve the object

### Scenario A: rename a resource block

Original address: aws_s3_bucket.app

New HCL resource label: aws_s3_bucket.application

Move the binding:

~~~bash
terraform state mv \
  aws_s3_bucket.app \
  aws_s3_bucket.application
~~~

Expected address change:

~~~text
Before: aws_s3_bucket.app
After:  aws_s3_bucket.application
~~~

Inspect:

~~~bash
terraform state list
terraform state show aws_s3_bucket.application
terraform plan
~~~

The plan should not propose destroying and recreating the bucket solely because its Terraform label changed.

### Scenario B: move a resource into a module

Old address: aws_s3_bucket.app

New address: module.app_bucket.aws_s3_bucket.this

~~~bash
terraform state mv \
  aws_s3_bucket.app \
  module.app_bucket.aws_s3_bucket.this
~~~

The destination module and resource configuration must exist and represent the same physical object.

### Prefer declarative moved blocks for code refactors

For a source-controlled refactor, a moved block is often more reviewable:

~~~hcl
moved {
  from = aws_s3_bucket.app
  to   = aws_s3_bucket.application
}
~~~

Run terraform plan and confirm Terraform recognizes an address move rather than destroy/create. Keep the moved block according to your team's compatibility policy so relevant state instances can migrate.

Use terraform state mv for controlled state surgery or when a moved block is unsuitable. Do not use both mechanisms for the same mapping in one migration.

## 5. terraform state rm — stop tracking without destroying

### Scenario

A bucket is being handed over to another Terraform root module or management system. The current state must stop managing it, but the bucket must remain in AWS.

Confirm the destination owner is ready and there is no overlapping active management. Back up state, then:

~~~bash
terraform state rm aws_s3_bucket.app
~~~

Inspect:

~~~bash
terraform state list
terraform plan
~~~

After removal, the address is no longer in this state. The real bucket remains in AWS.

**Critical consequence:** If the same resource block remains in this configuration, the next plan will normally propose creating a new bucket because Terraform sees a configured resource with no state binding. If ownership is transferring, remove/refactor the old resource block in the same reviewed change, or coordinate a state handoff to the new owner.

State removal is not a substitute for destroying an object. To destroy infrastructure, use a reviewed Terraform destroy workflow while the resource is still tracked.

## 6. Back up and inspect state before surgery

For local state:

~~~bash
terraform state pull > state-backup.json
~~~

Protect the backup as sensitive data. It can contain secrets and resource attributes. Store it only in an approved secure location; never commit it to Git.

For remote state, coordinate a maintenance window or deployment lock, use backend versioning and the team's approved backup/recovery procedure, and ensure no concurrent operation is running.

Useful commands:

~~~bash
terraform state list
terraform state show aws_s3_bucket.app
terraform state pull
terraform plan
terraform show
~~~

Use exact addresses from terraform state list. Addresses containing indexes or keys need shell quoting:

~~~bash
terraform state show 'aws_instance.app["api"]'
terraform state mv 'aws_instance.app[0]' 'aws_instance.app["api"]'
~~~

## 7. Failure — import points to the wrong object or address

Scenario: a developer imports the wrong bucket ID or tries to import an object into an address already tracked.

Safe response:
1. Stop and do not apply.
2. Verify AWS account and region with aws sts get-caller-identity and provider configuration.
3. Check the exact address using terraform state list.
4. Inspect the physical identifier using terraform state show.
5. Confirm the import ID using provider documentation and the actual AWS object.
6. If the wrong object was imported and no other operations occurred, coordinate a correction. Back up state, remove only the incorrect binding with terraform state rm, then import the correct object into the intended address.
7. Run a fresh plan and confirm no unintended create, update, replacement, or destroy actions.

Never import the same remote object into multiple Terraform addresses or states that will both manage it. That creates competing ownership.

## 8. Failure — state mv causes a destroy/create plan

Scenario: a resource was renamed in HCL, but the state address was not moved. Terraform plans to destroy the old address and create the new one.

Safe response:
1. Do not apply the destroy/create plan.
2. Compare old and new addresses and confirm they represent the same intended object.
3. Back up state and inspect the old binding with terraform state show.
4. Use a correct moved block or controlled terraform state mv.
5. Run terraform state list and confirm only the new address exists.
6. Run a fresh terraform plan.
7. Confirm Terraform recognizes the move and does not replace the physical object.

If the destination address is already occupied by a different object, stop. Resolve ownership and address collisions first; do not overwrite a mapping.

## 9. Failure — state rm followed by unexpected creation

Scenario: a resource was removed from state, but its resource block remains in configuration. A later plan proposes creating another object.

Safe response:
1. Do not apply.
2. Confirm the real object still exists and identify its current owner.
3. Confirm whether this Terraform configuration should still declare it.
4. If this state should continue managing it, re-import it at the correct address after checking that no other state owns it.
5. If ownership has transferred, remove/refactor the old resource block and ensure the new owner has a valid state binding.
6. Run a fresh plan and verify the intended ownership transition.

Do not delete the real resource just to make the plan empty.

## 10. Common mistakes

- Assuming import creates configuration or infrastructure.
- Using the wrong provider-specific import ID.
- Importing an object already tracked at another address.
- Renaming HCL without moving the state address.
- Moving state to a destination that represents a different object.
- Running state mv or state rm while another pipeline changes the same state.
- Treating state rm as a destroy command.
- Leaving the old resource block after state rm and being surprised by a create plan.
- Forgetting state backups can contain sensitive values.
- Applying a plan without checking physical IDs and exact addresses.

## 11. Interview-ready mental model

**Import adopts an existing object; state mv changes its Terraform address; state rm forgets the object without deleting it.** All three manipulate Terraform's inventory. Infrastructure changes only through provider operations in an approved plan/apply workflow, not merely because a state binding was moved or removed.
