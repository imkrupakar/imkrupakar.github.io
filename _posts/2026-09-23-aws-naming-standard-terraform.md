---
title: "AWS Has No Naming Standard. Here's a Small One You Can Build in Terraform"
date: 2026-09-23 14:13:00 +0530
categories: [Platform Engineering, Terraform]
tags: [aws, terraform, naming, azure, caf, platform-engineering]
description: A CAF-style naming pattern for AWS in plain Terraform, using Availability Zone IDs to derive region short codes instead of a hand-maintained lookup table.
media_subpath: /assets/img/posts/aws-naming-standard-terraform
---

## The problem

On Azure, naming is mostly solved. Microsoft's [Cloud Adoption Framework (CAF) naming guidance](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming) defines a standard, and the [Azure naming module for Terraform](https://registry.terraform.io/modules/Azure/naming/azurerm/latest) turns a few inputs into consistent resource names.

AWS has no equivalent. There is no official standard and no first-party naming module, so every team ends up with its own `"${var.project}-${var.env}-${var.region}"` string. The results are inconsistent: one team writes `us-east-1`, another writes `ue1`, and a hand-maintained region map goes stale.

This post shows a small, predictable naming pattern for AWS in plain Terraform. It relies on one trick that removes the region lookup table entirely.

## The trick: AWS already publishes region short codes

Long region names like `us-east-1` are awkward in resource names because of the length and the hyphens. You want something compact, like `use1`.

AWS already provides this. Every [Availability Zone ID](https://docs.aws.amazon.com/ram/latest/userguide/working-with-az-ids.html) begins with a short region code:

```text
use1-az1     → us-east-1
usw2-az3     → us-west-2
euw1-az2     → eu-west-1
apse1-az1    → ap-southeast-1
```

The part before the first hyphen is a stable region short code. You can read it at plan time with the [`aws_availability_zones`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones) data source:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  # "use1-az1" -> "use1"
  region_short = split("-", data.aws_availability_zones.available.zone_ids[0])[0]
}
```

That one local replaces a lookup table you would otherwise maintain by hand. New regions work automatically, because AWS assigns the AZ IDs itself.

![Overview of the AWS naming pattern: get the region short code from AZ IDs, define inputs, build the name and tags with compact(), add guardrails, use an override for legacy resources, and example names for RDS, S3, and Lambda](naming-pattern-overview.png)
*The whole pattern on one page: region short code, inputs, name and tags, guardrails, overrides, and examples.*

## Borrow CAF's structure, not its abbreviations

The useful part of CAF is the structure: a fixed order of name parts, all lowercase, and optional parts that disappear cleanly when unused. The specific abbreviations matter less, so you can pick your own.

Our pattern is:

```text
<service>-<workload>[-<purpose>]-<environment>-<region_short>[-<sequence>]
```

Examples, for an RDS database:

```text
rds-orders-prod-use1
rds-orders-reporting-prod-use1-001
```

The service part is an input, not something hardcoded, so the same logic names an S3 bucket, a Lambda function, or a database. Here are the inputs:

```hcl
variable "service" {
  type        = string
  description = "Short slug for the resource type, e.g. rds, s3, lambda."
}

variable "workload" {
  type        = string
  description = "Workload or application name, e.g. orders."
}

variable "purpose" {
  type        = string
  default     = null
  description = "Optional qualifier, e.g. reporting."
}

variable "environment" {
  type        = string
  description = "Environment name."

  validation {
    condition     = contains(["dev", "stg", "prod", "sbx"], var.environment)
    error_message = "environment must be one of: dev, stg, prod, sbx."
  }
}

variable "region" {
  type        = string
  description = "Full AWS region, e.g. us-east-1."
}

variable "sequence" {
  type        = number
  default     = null
  description = "Optional instance number, rendered as three digits."
}

variable "region_short_override" {
  type        = string
  default     = null
  description = "Use only if the derived region short code is wrong for an edge-case region."
}
```

Then the name is built in one place:

```hcl
locals {
  region_short = coalesce(
    var.region_short_override,
    split("-", data.aws_availability_zones.available.zone_ids[0])[0]
  )

  name_tokens = [
    for t in compact([
      var.service,
      var.workload,
      var.purpose,
      var.environment,
      local.region_short,
      var.sequence == null ? null : format("%03d", var.sequence),
    ]) : lower(t)
  ]

  resource_name = join("-", local.name_tokens)
}
```

The key function is [`compact()`](https://developer.hashicorp.com/terraform/language/functions/compact). It removes any null or empty values from the list, so `purpose` and `sequence` simply vanish when you don't set them. One expression produces both short and long names.

## Derive tags from the same inputs

Good naming usually comes with baseline tags. Since you already have the inputs, generate the tags from them so callers don't repeat themselves:

```hcl
locals {
  tags = merge(
    {
      workload    = var.workload
      environment = var.environment
      region      = var.region
      managedby   = "terraform"
    },
    var.tags # caller-supplied tags
  )
}
```

Notice the split: the **name** uses the short code (`use1`) because it reads well in an identifier, while the **tag** uses the full region (`us-east-1`) because that is what reporting and policy tools expect. It is one input in two forms.

## Add guardrails

Two checks prevent the most common mistakes, and both use Terraform's [custom conditions](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions).

**1. Make sure the declared region matches the provider's region.** Otherwise the name and tag could say one region while the resource is created in another. With AWS provider v6, the current region is `data.aws_region.current.region` (the older `.name` is deprecated; see the [v6 upgrade guide](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-6-upgrade)). It comes from the [`aws_region`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/region) data source.

**2. Check length limits.** Many AWS resources cap name length, and it is better to fail at plan time than at apply time.

```hcl
data "aws_region" "current" {}

resource "aws_db_instance" "this" {
  identifier = local.resource_name
  # ...

  lifecycle {
    precondition {
      condition     = var.region == data.aws_region.current.region
      error_message = "var.region (${var.region}) does not match the provider region (${data.aws_region.current.region})."
    }

    precondition {
      condition     = length(local.resource_name) <= 63
      error_message = "Derived name '${local.resource_name}' exceeds the 63-character RDS limit."
    }
  }
}
```

Also give yourself an escape hatch for resources that predate the standard. A `resource_name_override` input lets you keep an existing name when you import a legacy resource, instead of forcing a rename and a replacement:

```hcl
variable "resource_name_override" {
  type    = string
  default = null
}

locals {
  final_name = coalesce(var.resource_name_override, local.resource_name)
}
```

## Why not publish a shared naming module?

You could wrap all of this in a reusable module, as Azure does. We chose not to. The logic is about 50 lines, so we keep it in a `naming.tf` file inside each platform module. When a name comes out wrong, you can read the logic right there instead of tracing through another module and version. For something this small, a few readable locals are easier to maintain than a new dependency.

If you find yourself copying the file into many modules, that is the point where a shared module starts to make sense.

## Summary

- AWS has no official naming standard, but it does expose region short codes through Availability Zone IDs. Read them with `aws_availability_zones` instead of maintaining a map.
- Keep a fixed order of name parts, lowercase everything, and use `compact()` so optional parts drop out cleanly.
- Make the service slug an input so one naming block works for every resource type.
- Derive tags from the same inputs: short region in names, full region in tags.
- Add preconditions for region mismatch and length limits, and provide override inputs for legacy resources and unusual regions.

Azure gives you the full framework. On AWS you build a small version yourself, and one underused data source does most of the work.

---

*Also published on [Medium](https://medium.com/@krupakarreddy/caf-style-naming-on-aws-with-terraform-when-theres-no-avm-to-lean-on-ea85e10c7964).*
