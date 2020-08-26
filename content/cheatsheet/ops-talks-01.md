---
title: "Ops-Talks #01"
date: 2020-08-26T02:46:23+08:00
draft: false
tags: ["ops-talks-01", "terraform", "cheatsheet"]
categories: ["Syntax Highlighting"]
toc: false
type: "cheatsheet"
slide: https://slides.com/floriandambrine/ops-talks-01-terraform
---

### 1.0 Development

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

<!--more-->

### 2.0 Bypass Terragrunt module and use plain Terraform file instead

```hcl {linenos=table,hl_lines=["15-17"],linenostart=1}
### terragrunt.hcl
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

```hcl {linenos=table,linenostart=1}
### main.tf

variable "aws_region" {}

provider aws {
  region = var.aws_region
}

resource "..."
```
