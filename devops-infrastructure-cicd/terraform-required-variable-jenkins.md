---
layout: default
title: Terraform Required Variables and Jenkins
parent: Infrastructure & CI/CD
nav_order: 8
---

## Terraform Required Variables and Jenkins

**Scenario:** Jenkins runs `terraform plan -var-file qa.tfvars` and hangs waiting for input, then times out. The variable exists in `environment_variables` inside the tfvars file. Why isn't that enough?

Here is the setup that triggers the problem:

```hcl
# variables.tf
variable "region" {
  type    = string
  default = "eu-west-1"
}

variable "app_image" {
  type    = string
  default = "my-app:latest"
}

variable "environment_variables" {
  type = list(object({
    name  = string
    value = string
  }))
  default = []
}

variable "config_cache_ttl" {
  type = number          # no default - Terraform will prompt if not supplied
}
```

```hcl
# qa.tfvars
region    = "eu-west-1"
app_image = "my-app:3.4.1"

environment_variables = [
  { name = "APP_ENV",          value = "qa"  },
  { name = "LOG_LEVEL",        value = "debug" },
  { name = "CONFIG_CACHE_TTL", value = "3"   },
]

# config_cache_ttl is NOT assigned here as a top-level value
```

---

### Why Terraform Prompted for a Value

When you declare a variable in `variables.tf` without a default, Terraform treats it as required. It must be satisfied by one of:

- a `default` in the declaration
- a top-level assignment in a `.tfvars` file
- a `-var` flag on the CLI
- a `TF_VAR_` environment variable

```hcl
# variables.tf — no default means required
variable "config_cache_ttl" {
  type = number
}
```

When Jenkins ran `terraform plan -var-file qa.tfvars`, Terraform scanned all four sources, found nothing for `config_cache_ttl`, and fell back to prompting interactively:

```
var.config_cache_ttl
  Enter a value:
```

Jenkins can't answer that prompt, so the pipeline stalled and failed.

---

### Why `environment_variables` Didn't Help

The `environment_variables` block in the tfvars is a list of strings passed to ECS containers at runtime. Terraform has no idea these relate to the declared variable — they are completely separate concepts:

```hcl
# qa.tfvars
environment_variables = [
  { name = "CONFIG_CACHE_TTL", value = "3" },  # passes value to container
  # ...
]
```

This satisfies the `environment_variables` input variable (a list), not `config_cache_ttl` (a number). Terraform does not parse string values inside lists looking for matching variable names. The declared variable is still unsatisfied.

---

### The Fix

Two changes together:

**1. Add a default to the variable declaration**

```hcl
# variables.tf
variable "config_cache_ttl" {
  type    = number
  default = 10800
}
```

Now every environment that doesn't override it gets `10800`. Terraform never prompts.

**2. Add a top-level assignment in the environment tfvars**

```hcl
# qa.tfvars
config_cache_ttl = 3   # overrides the default for QA

environment_variables = [
  { name = "CONFIG_CACHE_TTL", value = "3" },
  # ...
]
```

The top-level assignment is what satisfies the Terraform variable. The entry inside `environment_variables` is what actually passes the value into the container. Both are needed and they serve different purposes.

---

### The Full Picture

| Thing | Purpose |
|---|---|
| `variable "config_cache_ttl"` in `variables.tf` | Declares the Terraform variable with default `10800` |
| `config_cache_ttl = 3` in `qa.tfvars` | Overrides it to `3` for QA |
| `{ name = "CONFIG_CACHE_TTL", value = "3" }` in `environment_variables` | Passes the value to the container as a runtime env var |

The Terraform variable and the container env var are currently separate — the variable isn't wired to set the env var (that would require using it in `main.tf`). It exists for documentation and to prevent Jenkins from being prompted again if the tfvars entry is ever removed.

---

### Preventing This Class of Problem

Any variable declaration without a default is a pipeline timebomb — it will block the next unattended run that doesn't supply it. Two rules help:

**Always add a default for infrastructure variables that have a sensible fallback.**

```hcl
variable "config_cache_ttl" {
  type        = number
  default     = 10800
  description = "TTL in seconds for the application config cache."
}
```

**Use `description` to document what the variable does** — especially when it is not wired directly to a resource and exists for other reasons (documentation, future use, consistency across environments). Future engineers won't have to guess why it's declared.
