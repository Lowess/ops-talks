---
title: "Ops-Talks #01"
date: 2020-08-26T02:46:23+08:00
draft: false
tags: ["ops-talks-01", "terraform", "cheatsheet"]
categories: ["Syntax Highlighting"]
toc: false
type: "cheatsheet"
slide: https://slides.com/floriandambrine/ops-talks-01-terraform
article:
  title: Stop being selfish - Open up Terraform to your team with Atlantis
  logo: medium.png
  link: https://medium.com/gumgum-tech/stop-being-selfish-open-up-terraform-to-your-team-with-atlantis-9b21f40fd73e
---

### 1.1 Terraform `state`

> Terragrunt repository layout dictates where its corresponding `terraform tfstate` file will be stored. Moving an existing component folder should result in moving it's states along to the corresponding path in S3.

* The main `terragrunt.hcl` file under the root of the repo uses a dynamic variable lookups to determine `aws_region` / `product_name` / `account_name`

<!--more-->

```hcl {linenos=table,hl_lines=["15-16"],linenostart=1}
### <repo>/terragrunt/terragrunt.hcl

locals {
  # Automatically load account-level variables
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))

  # Automatically load region-level variables
  region_vars = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  # Extract the variables we need for easy access
  account_name = local.account_vars.locals.account_name
  product_name = local.account_vars.locals.product_name
  aws_region   = local.region_vars.locals.aws_region

  remote_state_bucket        = "terraform-state-${local.product_name}-${local.account_name}-${local.aws_region}"
  remote_state_bucket_prefix = "${path_relative_to_include()}/terraform.tfstate"
}

remote_state {
  backend = "s3"
  config = {
    encrypt        = true
    bucket         = "${local.remote_state_bucket}"
    key            = "${local.remote_state_bucket_prefix}"
    region         = local.aws_region
    dynamodb_table = "terraform-locks"
  }
  ...
}
```

* Ran `terragrunt plan` from:
```
terragrunt/gumgum-ai/virginia/prod/mle/ecs-cluster/ecs/terragrunt.hcl
```
* Terragrunt `account.hcl` values:
```json
{
  "product_name": "verity",
  "account_name": "gumgum-ai",
  "aws_region": "us-east-1"
}
```

* Computed S3 storage:
```
s3://terraform-state-verity-gumgum-ai-us-east-1/gumgum-ai/virginia/prod/mle/ecs-cluster/ecs/terraform.tfstate
```

### 1.2 Repository Layout

| Path  | Description | Values |
|-------|-------------|--|
| `terraform/modules`       | Directory containing a list of [Terraform modules](https://www.terraform.io/docs/modules/index.html) (building blocks)             | `Must follow [a-z-]+ naming convention` |
| `terraform/terragrunt/<account>` | Project root directory for resources located and provisioned in the given `<account>` | `gumgum-ads`, `gumgum-ai`, `gumgum-sports` |
| `terraform/terragrunt/<account>/<region>` | Directory for resources provisioned in the given `<region>` | `virginia`, `oregon`, `ireland` |
| `terraform/terragrunt/<account>/<region>/<env>` | Directory for resources provisioned in the given `<env>` | `prod`, `stage`, `dev` |
| `terraform/terragrunt/<account>/<region>/<env>/<product>` | Directory for resources linked to a `<product>` | `verity-api`, `mle-inference` |
| `terraform/terragrunt/<account>/<region>/<env>/<product>/<component>` | Infrastructure `<component>` of a `<product>`. A `<component>` name is usually made of the AWS service it refers to or the software it installs followed by a `<component>` string identifier | `dynamodb-verity-pages`, `kafka-cluster`, `s3-coverage` |

---

### 1.3 Local development

```bash
# Create the `tf` alias that will spin up a docker container
# with all the tooling required to operate Verity infrastructure

export GIT_WORKSPACE="${HOME}/Workspace"

alias tf='docker run -it --rm \
    -v ~/.aws:/root/.aws \
    -v ~/.ssh:/root/.ssh \
    -v ~/.kube:/root/.kube \
    -v "$(pwd):/terragrunt" \
    -v "${GIT_WORKSPACE}:/modules" \
    -w /terragrunt \
    lowess/terragrunt:0.12.29'
```

### 1.4 Override Terragrunt module

```bash  {linenos=table,hl_lines=["9"],linenostart=1}
### Terminal Laptop
$ cd ${GIT_WORKSPACE}/verity-infra-ops/terraform/terragrunt
$ tf

### Entered docker container
/terragrunt $ cd <path-to-component-you-want-to-test>

### Plan with terragrunt module override (point to local)
/terragrunt $ terragrunt plan --terragrunt-source /modules/terraform-verity-modules//<somemodule>

```
---

### 2.1 Bypass Terragrunt module and use plain Terraform file instead

```hcl {linenos=table,hl_lines=["15-17"],linenostart=1}
# ---------------------------------------------------------------------------------------------------------------------
# TERRAGRUNT CONFIGURATION
# This is the configuration for Terragrunt, a thin wrapper for Terraform that supports locking and enforces best
# practices: https://github.com/gruntwork-io/terragrunt
# ---------------------------------------------------------------------------------------------------------------------

locals {
  # Automatically load environment-level variables
  environment_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))

  # Extract out common variables for reuse
  env = local.environment_vars.locals.environment
}

# Terragrunt will not attempt to fetch a module as the terraform block bellow is empty
# it will instead execute bare *.tf files found in the folder
terraform {}

# Include all settings from the root terragrunt.hcl file
include {
  path = find_in_parent_folders()
}

# These are the variables we have to pass in to use the module specified in the terragrunt configuration above
inputs = {}
```

* Create a `<filename>.tf` file for example `main.tf`:

```hcl {linenos=table,hl_lines=["0-99"]linenostart=1}
### main.tf

variable "aws_region" {}

provider aws {
  region = var.aws_region
}

resource "..."
```

---

### 2.2 Terragrunt provider overwrites

```hcl {linenos=table,hl_lines=["20-30"]linenostart=1}
# ---------------------------------------------------------------------------------------------------------------------
# TERRAGRUNT CONFIGURATION
# This is the configuration for Terragrunt, a thin wrapper for Terraform that supports locking and enforces best
# practices: https://github.com/gruntwork-io/terragrunt
# ---------------------------------------------------------------------------------------------------------------------

locals {
  # Automatically load environment-level variables
  environment_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  # Extract out common variables for reuse
  env = local.environment_vars.locals.environment
}

# Terragrunt will copy the Terraform configurations specified by the source parameter, along with any files in the
# working directory, into a temporary folder, and execute your Terraform commands in that folder.
terraform {
  source = "git::ssh://git@bitbucket.org/..."
}

generate "provider" {
  path = "providers.tf"
  if_exists = "overwrite_terragrunt"
  contents = <<EOF
    # Configure the AWS provider
    provider "aws" {
      version = "~> 2.9.0"
      region = var.aws_region
    }
  EOF
}

# Include all settings from the root terragrunt.hcl file
include {
  path = find_in_parent_folders()
}

# These are the variables we have to pass in to use the module specified in the terragrunt configuration above
inputs = {}
```

---

### 2.3 Versioning at scale

* How to use global module versioning from `terragrunt.hcl`

```hcl {linenos=table,hl_lines=["2-3", "7"]linenostart=1}
locals {
  versioning_vars = read_terragrunt_config(find_in_parent_folders("versioning.hcl"))
  version = lookup(local.versioning_vars.locals.versions, "mynamespace/app", "latest")
}

terraform {
  source = "/terraform-verity-modules//verity-api/dynamodb?ref=${local.version}"
}
```

* Add your versions in `terragrunt/versioning.hcl`:

```hcl {linenos=table,hl_lines=["6"]linenostart=1}
locals {
  versions = {
    "latest": "v1.7.2"
    "verity-api/dynamodb": "v1.4.2"
    "nlp/elasticache": "v1.7.2"
    "namespace/app": "v1.7.1"
  }
}
```
