---
title: "Putting a Kafka Topic Naming Convention into Practice with Terraform"
head_title: "Putting a Kafka Topic Naming Convention into Practice with Terraform"
description: "Putting a Kafka Topic Naming Convention into Practice with Terraform"
date: 2026-07-08
tags:
  - java
  - kafka
  - terraform
  - IAC
  - CI/CD
cascade:
  showDate: true
  showAuthor: true
  showSummary: true
  invertPagination: true
---

From gentle suggestions to enforced rules.
<!--more-->
---

I got tired of seeing Kafka topic naming conventions end up as wiki pages that everyone ignores
because in the end they are just gentle suggestions.

This article shows how to enforce a naming convention with Terraform or OpenTofu while at the same time creating those
Kafka resources in a clean and structured way.

This article builds on a [previous one](https://jonasg.io/posts/kafka-without-structure-is-just-kafka/) where I introduced
the idea of a topic naming convention.

> A simple convention will be used as an example, but the principles apply to any convention.
> The goal is to show how to enforce a naming convention with Terraform.

The *public* topics have exactly four segments:

```
<environment>.<domain>.<subdomain>.<type>
```

- The possible environments are `dev`, `uat`, and `prd`.
- The possible domains are `orders`, `inventory`, and `payments`.
- All characters are limited to lowercase letters and dashes.
- The type is either `event` or `state` where `state` topics **are compact with infinite retention**.

Some examples:

- `prd.orders.checkout.event`
- `dev.inventory.stock.state`
- `prd.orders.refund.state`

Application *internal* topics consist of:

```
<environment>.<type-of-service>.<app-name>[.<subject>]
```

Where type-of-service is one of: `app` (for domain applications), `tool` (for tools), `connect` (for kafka-connect).

Some examples:

- `dev.app.inventory-service.event`
- `prd.connect.source-s3-billing`

Even consumer groups can follow conventions, `<environment>.app.<app-name>[.<consumer-group-name>]`.

Some examples:

- `dev.app.inventory-service`
- `dev.app.inventory-service.api`

## The goal

```hcl

# The name of the service that owns the topics.
# Used to verify the internal topics naming and consumer group convention.
service_name = "inventory-service"

topics = [
  {
    name               = "${var.env}.inventory.stock.event"
    partitions         = var.partitions
    replication_factor = var.replication_factor
    retention_tier     = "long"
  },
  {
    name               = "${var.env}.inventory.stock.state"
    partitions         = var.partitions
    replication_factor = var.replication_factor
  },
  {
    name               = "${var.env}.app.inventory-service.belt"
    partitions         = var.partitions
    replication_factor = var.replication_factor
    retention_tier     = "short"
  },
]

consumer_groups = ["${var.env}.app.inventory-service"]
```

## Step 1: Wrap the Kafka provider resources

We will use the [Mongey/kafka](https://registry.terraform.io/providers/Mongey/kafka/latest/docs) provider, but the
[confluentinc/confluent](https://registry.terraform.io/providers/confluentinc/confluent/latest/docs) provider is also a
good option.
The wrapper module will be a thin layer that adds validation and governance rules.

`variables.tf` defines a custom convenience variable for the topics to be created.

```hcl
variable "topics" {
  type = list(object({
    name               = string
    partitions         = number
    replication_factor = number
    config = optional(map(string), {})
  }))
}
```

`main.tf` will be responsible for creating the topics using the underlying module.

```hcl
resource "kafka_topic" "this" {
  for_each = {for t in var.topics : t.name => t}

  name               = each.value.name
  partitions         = each.value.partitions
  replication_factor = each.value.replication_factor
  config             = each.value.config
}
```

Often overlooked yet extremely valuable is testing a module like this.

`main.tftest.hcl`:

```hcl
mock_provider "kafka" {}

run "create_topic" {
  variables {
    topics = [
      {
        name               = "orders"
        partitions         = 6
        replication_factor = 3
        config = {
          "cleanup.policy" = "compact"
          "retention.ms"   = "604800000"
        }
      },
    ]
  }

  assert {
    condition     = length(kafka_topic.this) == 1
    error_message = "Expected exactly 1 topic to be created"
  }

  assert {
    condition     = kafka_topic.this["orders"].name == "orders"
    error_message = "orders topic name does not match"
  }

  assert {
    condition     = kafka_topic.this["orders"].partitions == 6
    error_message = "orders partitions does not match"
  }
  // more assertions in the example repo
}
```

## Step 2: Add the validation rules

The attentive reader will have noticed that the created topic `orders` in our tests does **not** follow the convention.
Once the naming convention is validated, the test will fail with a clear message.

Maybe let's add the tests first?

```hcl
run "create_valid_topic" {
  variables {
    topics = [
      {
        name               = "prd.orders.checkout.event"
        partitions         = 6
        replication_factor = 3
        config = {
          "cleanup.policy" = "compact"
          "retention.ms"   = "604800000"
        }
      },
    ]
  }

  assert {
    condition     = length(kafka_topic.this) == 1
    error_message = "Expected exactly 1 topic to be created"
  }

  assert {
    condition     = kafka_topic.this["prd.orders.checkout.event"].name == "prd.orders.checkout.event"
    error_message = "topic name does not match"
  }
  // more assertions in the example repo
}
```

Now that we have a failing test, we can add the validation rules to the wrapper module.

```hcl
variable "topics" {
  // previous code left out for brevity
  validation {
    condition = alltrue([
      for t in var.topics :
      can(regex("^(dev|uat|prd)\\.(orders|inventory|payments)\\.[a-z-]+\\.(event|state)$", t.name))
    ])
    error_message = "Topic name must follow the convention <env>.<domain>.<subdomain>.<type> where env ∈ {dev,uat,prd}, domain ∈ {orders,inventory,payments}, subdomain is lowercase letters/dashes, type ∈ {event,state}."
  }
}
```

Variable blocks can contain multiple validation blocks, so we can add more rules to enforce the other segments of the
convention.

For example, imagine that we want to enforce that on `PRD` a minimum of 4 partitions is required.

```hcl
variable "topics" {
  // previous code left out for brevity
  validation {
    condition = alltrue([
      for t in var.topics : can(regex("^prd\\.", t.name)) ? t.partitions >= 4 : true
      ])
    error_message = "Topics in PRD must have at least 4 partitions."
  }
}
```

## Step 3: Add abstractions

At times, it helps to add abstractions to the module to make it easier to use and avoid a hotchpotch of different settings.
For example, we can introduce a `retention_tier` that abstracts away the `retention.ms` setting.
Who knows how many ms one month is anyway? This also avoids having all kinds of scattered or incoherent retention policies.

```hcl
locals {
  retention_tier_map = {
    short = "86400000"     # 1 day
    medium = "604800000"    # 7 days
    long = "2592000000"   # 30 days
    infinite = "-1"           # infinite retention
  }

  topic_configs = {
    for t in var.topics : t.name => merge(
        t.retention_tier != null ? { "retention.ms" = local.retention_tier_map[t.retention_tier] } : {},
      t.config,
        can(regex(".*\\.state$", t.name)) ? {
        "cleanup.policy" = "compact"
        "retention.ms"   = "-1"
      } : {},
    )
  }
}
```

And of course we can add a validation rule to ensure that the `retention_tier` is one of the allowed values and that
`config.retention.ms` and `retention_tier` are mutually exclusive.

```hcl
validation {
  condition = alltrue([
    for t in var.topics : t.retention_tier != null ? contains(["short", "medium", "long", "infinite"], t.retention_tier)
    : true
    ])
  error_message = "retention_tier must be one of: short, medium, long, infinite."
}

validation {
  condition = alltrue([
    for t in var.topics : !(t.retention_tier != null && can(t.config["retention.ms"]))
  ])
  error_message = "retention_tier and config.retention.ms are mutually exclusive, set only one."
}
```

Since the naming convention states that `state` topics are compact with infinite retention, we enforce that too.
The merge order in `topic_configs` ensures `.state` topics always get `cleanup.policy = "compact"` and
`retention.ms = "-1"`, and validation blocks catch conflicting input early:

```hcl
validation {
  condition = alltrue([
    for t in var.topics : can(regex(".*\\.state$", t.name)) && t.retention_tier != null ? t.retention_tier == "infinite"
    : true
    ])
  error_message = "State topics use infinite retention automatically — retention_tier must be 'infinite' or omitted."
}

validation {
  condition = alltrue([
    for t in var.topics : can(regex(".*\\.state$", t.name)) && can(t.config["retention.ms"]) ?
    t.config["retention.ms"] == "-1" : true
    ])
  error_message = "State topics use infinite retention automatically — config.retention.ms must be '-1' or omitted."
}

validation {
  condition = alltrue([
    for t in var.topics : can(regex(".*\\.state$", t.name)) && can(t.config["cleanup.policy"]) ?
    t.config["cleanup.policy"] == "compact" : true
    ])
  error_message = "State topics use cleanup.policy = compact automatically — config.cleanup.policy must be 'compact' or omitted."
}
```

The same abstraction principle applies to internal topic naming. A `service_name` variable lets callers declare their
service name, and the module checks that internal topics follow the `<env>.<type>.<app-name>[.<subject>]` convention
where `type` is `app`, `tool`, or `connect`:

```hcl
variable "service_name" {
  type    = string
  default = null
  validation {
    condition     = var.service_name == null || can(regex("^[a-z-]+$", var.service_name))
    error_message = "service_name must be null or consist of lowercase letters and dashes only."
  }
}
```

`service_name` also enables consumer group management. When you pass a list of consumer group names via
`consumer_groups`, the module creates `kafka_acl` resources with `resource_type = "Group"`, granting `Read` access to
`User:<service_name>`. This means naming and access control for consumer groups is handled alongside your topics.

By now you should have gotten the idea.

The full example module is available
at [https://github.com/jonas-grgt/kafka-conventions-demo-module](https://github.com/jonas-grgt/kafka-conventions-demo-module).
