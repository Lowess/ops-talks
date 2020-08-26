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
