# Terraform Interview Prep — Questions & Answers

> Interview-ready answers with AWS grounding. Answers are written the way you'd
> actually say them out loud: a crisp definition first, then the "why it matters"
> that shows production experience.

## Contents

1. [Basics](#1-basics)
2. [State](#2-state)
3. [Variables and Outputs](#3-variables-and-outputs)
4. [Modules](#4-modules)
5. [Expressions and Meta-Arguments](#5-expressions-and-meta-arguments)
6. [Provisioners](#6-provisioners)
7. [Backends and Workspaces](#7-backends-and-workspaces)
8. [Advanced](#8-advanced)
9. [AWS-Specific](#9-aws-specific)
10. [CI/CD Integration](#10-cicd-integration)
11. [Production Scenarios](#11-production-scenarios)

---

## 1. Basics

### What is Terraform?

An open-source Infrastructure as Code tool from HashiCorp that lets you declare
infrastructure in HCL, then creates, updates, and deletes real resources to match
that declaration. It's **declarative** (you describe the desired end state, not the
steps), **provider-based** (2000+ providers: AWS, GCP, Azure, Cloudflare, GitHub,
Datadog), and **stateful** (it records what it built in a state file so it can
compute diffs).

The mental model: `desired state (code)` vs `actual state (cloud API)` vs
`recorded state (state file)`. Every `plan` is a three-way diff between those.

### What are the advantages of using Terraform?

- **Cloud-agnostic** — one workflow and language across AWS, GCP, Azure, Cloudflare,
  Kubernetes, and SaaS tools. My Cloudflare DNS/CDN and AWS ECS resources live in the
  same repo.
- **Reviewable changes** — `terraform plan` gives a diff you can attach to a PR.
  Nobody merges infra blind.
- **Reproducible** — spin up an identical staging environment from the same modules.
- **Drift detection** — plan tells you when someone hand-edited the console.
- **Modularity and reuse** — versioned modules act like internal infra libraries.
- **Dependency graph** — Terraform figures out ordering and parallelism for you.
- **Large ecosystem** — providers, registry modules, tooling (tflint, Checkov, Infracost).

### What is Infrastructure as Code (IaC)?

Managing and provisioning infrastructure through machine-readable definition files
kept in version control instead of manual console clicks or ad-hoc scripts. Benefits:
version history, peer review, repeatability, automated testing, and disaster recovery
(the repo *is* the runbook).

Two flavors worth naming:

- **Declarative** (Terraform, CloudFormation, Pulumi) — describe the end state.
- **Imperative/procedural** (Ansible, shell, boto3 scripts) — describe the steps.

Also useful to distinguish **provisioning** (Terraform: create the VPC, the RDS
instance) from **configuration management** (Ansible: install packages on a host).

### How does Terraform differ from CloudFormation?

| | Terraform | CloudFormation |
|---|---|---|
| Scope | Multi-cloud + SaaS providers | AWS only |
| Language | HCL (plus JSON) | YAML/JSON |
| State | You manage it (S3 + DynamoDB) | AWS manages it for you |
| Preview | `terraform plan` | Change sets |
| Rollback | No auto-rollback; you fix forward | Automatic stack rollback on failure |
| Modularity | Modules, registry | Nested stacks, StackSets, modules |
| Drift | `plan` / `-refresh-only` | Drift detection API |
| Import | `import` block / `terraform import` | Resource import |

Practical take: CloudFormation wins on managed state and automatic rollback and is
sometimes required for AWS-native things like Service Catalog or SAM. Terraform wins
on multi-provider reach, expressiveness, and the plan/review workflow — which is why
it's the default choice when your stack spans AWS *and* Cloudflare *and* GitHub.

### What are Terraform providers?

Plugins that translate Terraform's resource model into API calls for a specific
platform. They're versioned binaries downloaded during `terraform init` into
`.terraform/providers`, and pinned in `.terraform.lock.hcl`.

```hcl
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.60"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

# Aliased second instance for a different region
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1" # e.g. ACM certs for CloudFront
}
```

Always pin provider versions — an unpinned major bump can rewrite your plan.

### What is a Terraform resource?

The fundamental building block: a declaration of one piece of infrastructure that
Terraform will create and own the lifecycle of.

```hcl
resource "aws_ecr_repository" "api" {
  name                 = "stitched-api"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

`aws_ecr_repository` is the type (provider prefix + resource), `api` is the local
name, and `aws_ecr_repository.api` is the address used in references and state.

### What is the difference between a provider and a resource?

The provider is the *plugin/client* — it knows how to authenticate and talk to an API.
The resource is a *single managed object* exposed by that provider. One provider
configuration manages many resources. Analogy: the provider is the AWS SDK client,
the resource is a specific object it manages.

### What is a data source in Terraform?

A read-only lookup of something Terraform does **not** manage. It fetches attributes
at plan time so you can reference them without hardcoding.

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/api/database"
}

resource "aws_security_group" "api" {
  vpc_id = data.aws_vpc.default.id
}
```

Key distinction: `resource` = "make this exist and own it"; `data` = "go find this and
tell me about it." Data sources also help decouple stacks — one stack can read another's
outputs via `terraform_remote_state` or by looking up tags/SSM parameters.

### What is the purpose of the terraform init command?

Initializes a working directory. It:

1. Downloads/installs required provider plugins.
2. Downloads modules (registry, Git, local).
3. Configures the backend and, if needed, migrates state.
4. Writes/updates `.terraform.lock.hcl`.

It's safe to re-run and idempotent. Useful flags: `-upgrade` (re-resolve versions),
`-backend-config=...` (inject backend settings in CI), `-reconfigure` and
`-migrate-state` (when changing backends).

### What does terraform plan do?

Refreshes state against the real API, compares it to your configuration, and prints
the execution plan — what will be created, updated in place, replaced, or destroyed —
without changing anything. It's the review artifact.

```bash
terraform plan -out=tfplan          # save a binary plan
terraform show -json tfplan | jq    # machine-readable, good for policy checks
```

Watch for the symbols: `+` create, `-` destroy, `~` update in place,
`-/+` destroy-then-create, `+/-` create-then-destroy, `<=` read (data source).

### What is the difference between terraform plan and terraform apply?

`plan` is read-only and produces the proposed change set. `apply` executes it. In CI
the right pattern is `plan -out=tfplan` on PR, then `apply tfplan` after approval —
applying the *saved* plan guarantees you execute exactly what was reviewed. A bare
`terraform apply` re-plans and prompts interactively (or skips the prompt with
`-auto-approve`, which should never be used against production without a gate).

### What does terraform destroy do?

Destroys everything in the current state for that configuration/workspace. It's
equivalent to `terraform apply -destroy`. Useful for ephemeral environments; dangerous
anywhere else. Guardrails: `prevent_destroy` lifecycle on critical resources, no
`-auto-approve` in pipelines, and IAM policies that deny delete on prod. To target a
subset (rarely a good idea): `terraform destroy -target=module.qa`.

### What is the purpose of terraform validate?

Checks the configuration for internal consistency — syntax errors, unsupported
arguments, wrong types, bad references. It does **not** contact any provider API or
need credentials, so it's a fast pre-commit / early-CI check. It does require
`terraform init` first (it needs provider schemas).

### What is terraform fmt?

Rewrites configuration files to canonical HCL formatting (indentation, alignment,
spacing). In CI use `terraform fmt -check -recursive` so formatting drift fails the
build instead of generating noisy diffs later.

### What does terraform refresh do?

Updates the state file to match what actually exists in the cloud, without changing
infrastructure. It's deprecated as a standalone command; the modern equivalent is:

```bash
terraform apply -refresh-only
```

Refresh happens implicitly at the start of every plan/apply anyway (disable with
`-refresh=false` for a faster plan on huge states). The `-refresh-only` mode is how you
deliberately reconcile drift into state and see what changed outside Terraform.

---

## 2. State

### What is the Terraform state file?

A JSON document (`terraform.tfstate`) mapping your configuration addresses to real
resource IDs, along with cached attribute values, resource dependencies, and the state
schema version. It's Terraform's memory of what it created.

### Why is the state file important?

- It's the **mapping** between code (`aws_instance.web`) and reality (`i-0abc123`).
  Without it Terraform doesn't know that resource is already yours and would try to
  create a duplicate.
- It stores **dependency metadata** needed for correct destroy ordering.
- It caches attributes so plans don't need to re-read everything.
- It's the coordination point for **locking** across a team.

It's also sensitive: it can contain database passwords, private keys, and tokens in
plaintext. Treat it like a secret — encrypted at rest, access-controlled, never in Git.

### Where is the Terraform state stored by default?

Locally, in `terraform.tfstate` in the working directory (with `terraform.tfstate.backup`
as the previous version). Fine for a scratch experiment, unacceptable for a team: no
locking, no durability, no sharing, and a real risk of committing secrets to Git.

### What is a remote backend?

A backend configuration that stores state outside your machine — S3, Azure Blob, GCS,
Terraform Cloud/HCP, Consul, Postgres — and usually provides locking. Init-time only;
backend blocks can't use variables (use partial configuration instead).

```hcl
terraform {
  backend "s3" {
    bucket         = "gl-terraform-state"
    key            = "stitched-health/prod/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    use_lockfile   = true   # native S3 locking, TF >= 1.10
    # dynamodb_table = "terraform-locks"  # classic approach
  }
}
```

### Why should you use a remote backend?

- **Shared state** — the whole team and CI see the same truth.
- **Locking** — prevents concurrent applies corrupting state.
- **Durability and versioning** — S3 versioning gives you point-in-time recovery.
- **Encryption + IAM** — secrets in state stay protected.
- **No state on laptops** — CI can apply without a developer's local file.

### How do you store Terraform state in AWS?

S3 bucket for the state object, with:

- **Versioning enabled** (recovery from bad writes).
- **SSE-KMS encryption** and a bucket policy denying unencrypted/non-TLS access.
- **Public access blocked**, and ideally a separate hardened account.
- **Locking** — `use_lockfile = true` (Terraform 1.10+, uses an S3 conditional-write
  lock file) or a DynamoDB table with primary key `LockID` for older versions.
- **One key per environment/stack**: `<project>/<env>/terraform.tfstate`.

Bootstrapping the bucket itself is a chicken-and-egg problem — create it once with a
small separate config (local state, then migrate) or via CLI/CloudFormation.

### Why is DynamoDB used with an S3 backend?

S3 historically had no locking primitive, so Terraform used a DynamoDB table
(`LockID` partition key) to implement a distributed mutex plus a state digest check.
Before writing, Terraform conditionally puts a lock item; a second apply fails fast
with "Error acquiring the state lock" instead of clobbering the first. Since Terraform
1.10, S3 conditional writes (`use_lockfile = true`) can do this natively and DynamoDB
is optional.

### What happens if two engineers run Terraform simultaneously?

With locking: the second run blocks or fails with a lock error showing who holds it,
when, and the lock ID. That's the desired outcome.

Without locking: both refresh, both compute plans from the same starting point, and the
last writer wins — the earlier writer's resources are silently dropped from state,
becoming orphaned/untracked infrastructure. Recovering means importing or restoring
from an S3 version. This is the single strongest argument for remote state.

### What is state locking?

A mutex Terraform acquires for the duration of any state-mutating operation
(`apply`, `destroy`, `state mv`, `import`). Supported by S3 (lockfile), DynamoDB,
Terraform Cloud, Consul, and Postgres backends; local files use OS file locks.

`-lock-timeout=5m` makes CI wait rather than fail. `terraform force-unlock <LOCK_ID>`
clears a stale lock left by a crashed run — only after confirming no apply is actually
running, or you'll cause the exact corruption locking prevents.

### How do you recover a corrupted state file?

1. **Stop all applies** and communicate — no one runs Terraform until it's fixed.
2. **Restore from S3 versioning** — list object versions, download the last good one,
   re-upload it as current. This is the fastest path and the reason versioning is
   mandatory.
3. **Local backup** — `terraform.tfstate.backup` if it happened locally.
4. **Surgical repair** — `terraform state pull > fixed.tfstate`, edit carefully (bump
   `serial`), `terraform state push fixed.tfstate`. Last resort.
5. **Rebuild by import** — if state is unrecoverable, write/keep the config and
   `terraform import` (or `import` blocks) each real resource back in.
6. **Verify** with `terraform plan` — a clean "no changes" means you're whole again.

Then fix the cause: enable locking, restrict who can write to the bucket, use CI-only
applies.

### What is the difference between terraform state and terraform import?

`terraform state` is a family of subcommands for inspecting and manipulating existing
state entries (`list`, `show`, `mv`, `rm`, `pull`, `push`, `replace-provider`).
`terraform import` *adds* a pre-existing, unmanaged real resource into state so
Terraform starts managing it. One reorganizes what Terraform already knows; the other
brings something new under management.

### What is terraform state rm?

Removes a resource from state without touching the real infrastructure — Terraform
"forgets" it and stops managing it. Use it when handing a resource to another
stack/module, when a resource was deleted out-of-band, or before re-importing at a
different address. Note the side effect: the config still declares it, so the next plan
will try to **create** it again unless you also remove the config. Modern alternative:
a `removed` block, which is declarative and reviewable.

### What is terraform state mv?

Renames/moves a state entry so Terraform recognizes a refactor instead of planning a
destroy-and-recreate.

```bash
terraform state mv aws_instance.web aws_instance.api
terraform state mv 'aws_instance.web[0]' 'aws_instance.web["primary"]'   # count -> for_each
terraform state mv module.old_vpc module.network
```

Since Terraform 1.1 the better approach is a `moved` block committed to the repo — it's
version-controlled, applies for everyone, and needs no manual CLI step:

```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.api
}
```

### What happens if the state file is deleted?

Terraform loses all knowledge of what it built. The infrastructure keeps running, but
the next plan proposes creating everything from scratch — and applying it either creates
duplicates or fails on name conflicts (S3 buckets, IAM roles). Recovery is S3 version
restore if you have it, otherwise a painful import of every resource. Prevention:
remote backend, versioning, MFA-delete or a deny-delete bucket policy, and no local
state for anything real.

---

## 3. Variables and Outputs

### What are Terraform variables?

Input parameters that make a configuration reusable — declared with a `variable` block,
referenced as `var.name`.

```hcl
variable "instance_type" {
  description = "EC2 instance type for the API tier"
  type        = string
  default     = "t3.small"

  validation {
    condition     = startswith(var.instance_type, "t3.")
    error_message = "Only t3 family is approved for this tier."
  }
}
```

Good practice: always set `description` and `type`, add `validation` for anything
foot-gun-shaped, and mark secrets `sensitive = true`.

### What are variable types?

**Primitives:** `string`, `number`, `bool`.

**Collections:** `list(T)`, `set(T)`, `map(T)`.

**Structural:** `object({...})`, `tuple([...])`.

Plus `any` (no constraint) and `optional()` attributes with defaults inside objects:

```hcl
variable "services" {
  type = map(object({
    cpu      = number
    memory   = number
    replicas = optional(number, 2)
  }))
}
```

### How do you pass variables to Terraform?

In precedence order, **lowest to highest**:

1. `default` in the variable block
2. `TF_VAR_name` environment variables
3. `terraform.tfvars` / `terraform.tfvars.json`
4. `*.auto.tfvars` (alphabetical order)
5. `-var-file=prod.tfvars` and `-var="key=value"` on the command line (last wins)

```bash
terraform apply -var-file=envs/prod.tfvars -var="image_tag=$GITHUB_SHA"
export TF_VAR_db_password="..."   # how CI passes secrets
```

Interactive prompt happens only for variables with no default and no supplied value.

### What is the difference between default and required variables?

A variable with a `default` is optional — omit it and the default applies. A variable
with no default is required; Terraform prompts interactively, or errors out in
`-input=false` automation. Use required variables deliberately for things that must
never be silently wrong (environment name, account ID, VPC ID) and defaults for sane
knobs.

### What are output values?

Exported values from a module or root configuration — the module's public API, and
the way you surface useful identifiers after apply.

```hcl
output "alb_dns_name" {
  description = "Public DNS of the ALB"
  value       = aws_lb.this.dns_name
}
```

Read them with `terraform output`, `terraform output -json`, or `-raw` for a single
value in scripts. Root outputs are also readable by other stacks via
`terraform_remote_state`.

### How do you mark outputs as sensitive?

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

Terraform then prints `(sensitive value)` in plan/apply output and refuses to display it
via plain `terraform output`. Caveats worth stating in an interview: it is **still
plaintext in state**, and `terraform output -json` will reveal it. It's log hygiene, not
encryption. The real answer for secrets is to keep them in Secrets Manager/SSM and output
the ARN, not the value.

### What are local values?

Named expressions computed once and reused — `locals` blocks, referenced as `local.name`.

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = "devops"
  }
}
```

They keep DRY things DRY (naming conventions, tag maps, computed CIDRs) without
exposing knobs callers shouldn't touch.

### What is the difference between variables and locals?

Variables are **inputs** — set from outside, can't reference resources or other
variables' computed results, and form the module's interface. Locals are **internal
derived values** — not settable from outside, but they can reference variables,
resources, data sources, and other locals. Rule of thumb: if the caller should decide
it, it's a variable; if it's computed from other things, it's a local.

---

## 4. Modules

### What is a Terraform module?

Any directory containing `.tf` files. It's a reusable, parameterized package of
resources with defined inputs (variables) and outputs. Every configuration has a root
module; anything it calls is a child module.

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"

  name = "${local.name_prefix}-vpc"
  cidr = "10.20.0.0/16"
  # ...
}
```

Sources can be local paths, the Terraform Registry, Git/SSH URLs, S3, or GCS.

### Why should you use modules?

- **Reuse** — one ECS-service module used by five services instead of five copies.
- **Consistency** — encryption, tagging, and logging baked in by default, so every
  bucket is compliant by construction.
- **Abstraction** — callers pass five inputs instead of understanding forty arguments.
- **Blast-radius control** — smaller, composable stacks.
- **Reviewability** — changing the module changes every consumer predictably, and
  versioning lets you roll it out gradually.

Caution: don't over-abstract. A module that's a thin passthrough wrapper around one
resource usually adds indirection without value.

### What is the difference between the root module and child modules?

The root module is the directory where you run `terraform` — it owns the backend
configuration, provider configurations, and the state file, and its outputs are what
`terraform output` shows. Child modules are called via `module` blocks; they inherit
providers (or receive them explicitly through `providers = {}`), cannot define a
backend, and communicate only through inputs and outputs.

### How do you pass variables to a module?

As arguments in the `module` block, matching the child's `variable` names:

```hcl
module "ecs_service" {
  source = "./modules/ecs-service"

  name          = "stitched-api"
  cluster_arn   = module.ecs_cluster.arn
  image         = "${aws_ecr_repository.api.repository_url}:${var.image_tag}"
  desired_count = var.environment == "prod" ? 3 : 1
  subnet_ids    = module.vpc.private_subnets
  tags          = local.common_tags
}
```

Values can be literals, variables, locals, or other modules' outputs — that's how you
wire a stack together.

### How do you use outputs from a module?

Reference them as `module.<name>.<output>`. The child must explicitly declare an
`output` block — internal resource attributes aren't reachable from the parent.

```hcl
output "service_url" {
  value = module.ecs_service.alb_dns_name
}
```

Referencing a module output also creates the dependency edge, so ordering is handled
automatically.

### How do you version Terraform modules?

- **Registry modules:** `version = "~> 5.8"` (pessimistic constraint — allows patches
  and minors, blocks the next major).
- **Git modules:** pin to a tag or commit SHA —
  `source = "git::https://github.com/org/tf-modules.git//ecs-service?ref=v1.4.2"`.
  A tag is readable; a SHA is immutable.
- Publish modules with **semantic versioning**: patch for fixes, minor for
  backward-compatible inputs, major for breaking input/behavior changes.
- Keep a `CHANGELOG.md` and never move a released tag.

Never leave a module unpinned on `main` — a teammate's merge shouldn't change your
production plan.

### How do you organize modules in a production project?

Split by **layer + lifecycle**, and keep environments as separate root modules with
separate state files:

```
infrastructure/
├── modules/                 # reusable building blocks (no backend, no env logic)
│   ├── vpc/
│   ├── ecs-cluster/
│   ├── ecs-service/
│   ├── rds/
│   └── iam-oidc-role/
├── live/
│   ├── dev/
│   │   ├── network/         # own state
│   │   ├── data/            # own state (RDS, ElastiCache) — slow-moving
│   │   └── apps/            # own state (ECS services) — fast-moving
│   ├── staging/
│   └── prod/
└── global/
    ├── iam/
    └── route53/
```

Principles: separate state per environment (a bad dev apply can't reach prod), split
long-lived infra from fast-changing app infra, no cross-environment references except
through explicit remote-state reads or SSM/tag lookups, and internal modules versioned
in their own repo once more than one team consumes them.

---

## 5. Expressions and Meta-Arguments

### What is count?

A meta-argument creating N instances of a resource or module, indexed numerically and
addressed as `resource.name[0]`.

```hcl
resource "aws_instance" "worker" {
  count         = var.worker_count
  ami           = var.ami_id
  instance_type = "t3.medium"
  tags          = { Name = "worker-${count.index}" }
}
```

Also the idiomatic conditional-create pattern:

```hcl
resource "aws_cloudwatch_log_group" "debug" {
  count = var.enable_debug_logs ? 1 : 0
  name  = "/aws/app/debug"
}
```

### What is for_each?

Creates one instance per element of a map or set of strings, keyed by a stable string
rather than a position. Inside the block you get `each.key` and `each.value`.

```hcl
resource "aws_ssm_parameter" "config" {
  for_each = {
    api_url   = "https://api.example.com"
    log_level = "info"
  }

  name  = "/stitched/prod/${each.key}"
  type  = "String"
  value = each.value
}
```

Iterating objects is the common production pattern:

```hcl
module "service" {
  source   = "./modules/ecs-service"
  for_each = var.services   # map(object({...}))

  name   = each.key
  cpu    = each.value.cpu
  memory = each.value.memory
}
```

### What is the difference between count and for_each?

| | `count` | `for_each` |
|---|---|---|
| Input | number | map or set of strings |
| Address | `res.name[0]` | `res.name["api"]` |
| Removing a middle item | reindexes → recreates everything after it | only that key is destroyed |
| Access | `count.index` | `each.key`, `each.value` |
| Best for | identical clones, on/off toggles | distinct named things |

The reindexing behavior is the reason to prefer `for_each` for anything non-trivial:
deleting the second of five subnets defined with `count` will destroy and recreate
subnets 2–5. With `for_each` keyed by AZ name, only the removed one goes.

### When would you use dynamic blocks?

When the *number of nested blocks* inside a resource is data-driven — security group
rules, ECS container port mappings, ALB listener rule conditions, IAM policy statements.

```hcl
resource "aws_security_group" "api" {
  name   = "api-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidrs
      description = ingress.key
    }
  }
}
```

Use sparingly — dynamic blocks hurt readability fast. If two or three static blocks
would do, write them out. (Also note SG rules are better modeled with the separate
`aws_vpc_security_group_ingress_rule` resources plus `for_each`.)

### What is the purpose of depends_on?

To declare a dependency Terraform can't infer. Terraform builds its graph from *value
references*, so most ordering is automatic. `depends_on` is for hidden/behavioral
dependencies — an IAM role policy that must exist before an ECS task can pull from ECR,
a NAT gateway that must be live before instances try to reach the internet, an
`aws_iam_service_linked_role`.

```hcl
resource "aws_ecs_service" "api" {
  # ...
  depends_on = [aws_iam_role_policy_attachment.task_execution]
}
```

Overusing it serializes the graph and slows applies. If you can express the dependency
by referencing an attribute instead, do that.

### What is the lifecycle block?

A per-resource meta-argument controlling how Terraform handles replacement and diffs:

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy       = true
    create_before_destroy = false
    ignore_changes        = [engine_version]

    precondition {
      condition     = var.environment != "prod" || var.multi_az
      error_message = "Production RDS must be Multi-AZ."
    }
  }
}
```

Members: `create_before_destroy`, `prevent_destroy`, `ignore_changes`,
`replace_triggered_by`, and `precondition` / `postcondition` checks.

### What does create_before_destroy do?

Inverts replacement order: build the new resource first, then destroy the old one.
Essential for zero-downtime replacement of launch templates/ASG instances, target
groups, and anything fronting live traffic.

```hcl
lifecycle {
  create_before_destroy = true
}
```

Constraint: the resource must tolerate two copies existing briefly, so unique names
collide. Use `name_prefix` instead of `name`, or a random suffix. Note it also
propagates to dependencies, which can widen the replacement set.

### What does prevent_destroy do?

Makes Terraform **error out** rather than destroy the resource. A safety pin for RDS
instances, S3 buckets holding data, KMS keys, and Route 53 zones.

```hcl
lifecycle {
  prevent_destroy = true
}
```

Important nuances: it blocks `terraform destroy` for that resource and blocks
destroy-then-create replacements, but it doesn't protect against someone removing the
lifecycle block in the same PR, and it doesn't stop deletion outside Terraform. Pair it
with cloud-side protection (`deletion_protection = true` on RDS/ALB, S3 versioning +
bucket policy) and IAM deny rules.

### What is ignore_changes?

Tells Terraform to stop diffing specific attributes after creation — for values legitimately
managed elsewhere.

```hcl
lifecycle {
  ignore_changes = [
    desired_count,   # managed by Application Auto Scaling
    task_definition, # rolled by the CI/CD pipeline
    tags["LastDeployed"],
  ]
}
```

`ignore_changes = all` exists but should be a last resort — it hides real drift.
This meta-argument is exactly what you reach for when the deploy pipeline (GitHub Actions
updating an ECS task definition) and Terraform both touch the same resource: Terraform
owns the shape, the pipeline owns the image tag.

### What is the purpose of replace_triggered_by?

Forces replacement of a resource when *another* resource or attribute changes — a
declarative way to express "if X changes, rebuild Y," replacing the old
`taint`/`null_resource` triggers hack.

```hcl
resource "aws_ecs_task_definition" "api" { /* ... */ }

resource "aws_ecs_service" "api" {
  # ...
  lifecycle {
    replace_triggered_by = [aws_ecs_task_definition.api]
  }
}
```

References must be to resources, whole instances, or attributes — not arbitrary
expressions or variables.

---

## 6. Provisioners

### What are Terraform provisioners?

Escape hatches that run scripts or copy files as part of resource creation or
destruction: `local-exec`, `remote-exec`, `file`, and (deprecated) vendor provisioners.

```hcl
resource "aws_instance" "app" {
  # ...
  provisioner "remote-exec" {
    inline = ["sudo systemctl restart app"]
    connection {
      type        = "ssh"
      host        = self.public_ip
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
    }
  }
}
```

They also support `when = destroy` and `on_failure = continue | fail`.

### What is the difference between local-exec and remote-exec?

`local-exec` runs a command on the machine running Terraform (your laptop or the CI
runner) — e.g. calling a CLI, invoking Ansible, writing a kubeconfig. `remote-exec`
runs commands *on the created remote resource* over SSH/WinRM, so it needs network
reachability, credentials, and a `connection` block.

### Why are provisioners generally discouraged?

HashiCorp itself calls them a last resort, because:

- **Not declarative** — Terraform can't plan them; you see no diff for what a script will do.
- **Not idempotent** — reruns can do damage or nothing at all.
- **No state tracking** — Terraform records that it ran, not what it changed.
- **Fragile failure mode** — a failed provisioner marks the resource *tainted*, so the
  next apply destroys and recreates it.
- **Connectivity and secrets coupling** — needs SSH keys, open ports, bastion access.
- **`when = destroy` runs don't always fire** (e.g. if the resource is already gone).

### What alternatives exist to provisioners?

- **Bake images** — Packer/EC2 Image Builder golden AMIs; nothing to configure at boot.
- **`user_data` / cloud-init** — declarative bootstrap owned by the instance itself.
- **Containers** — put the setup in the Dockerfile; that's the ECS/EKS answer.
- **Config management** — Ansible/Chef run as a separate pipeline stage after apply.
- **Real providers** — use the `kubernetes`, `helm`, `github`, or `cloudflare` provider
  instead of shelling out to a CLI.
- **`terraform_data` / `null_resource`** only when you genuinely need a local side effect.
- **AWS-native** — SSM Run Command / State Manager / Systems Manager Automation.

---

## 7. Backends and Workspaces

### What is a backend in Terraform?

The component that determines **where state is stored** and how state operations
(locking, reads, writes) are performed. Configured in the `terraform` block, resolved at
`init` time only, and it cannot reference variables or locals.

Use **partial configuration** to keep environment-specific values out of code:

```hcl
terraform {
  backend "s3" {}
}
```

```bash
terraform init \
  -backend-config="bucket=gl-terraform-state" \
  -backend-config="key=stitched/prod/terraform.tfstate" \
  -backend-config="region=ap-south-1"
```

Terraform 1.x also distinguishes state backends from HCP Terraform's `cloud` block,
which additionally provides remote *execution*.

### What backend types have you used?

Frame this around real usage:

- **local** — throwaway experiments and module development only.
- **s3** — the production default: versioned, KMS-encrypted bucket, DynamoDB or S3
  lockfile locking, one key per environment, cross-account access via an assumed role.
- **HCP Terraform / Terraform Cloud** — when you want managed state, remote runs, policy
  as code (Sentinel/OPA), and a UI-based approval workflow.
- **gcs / azurerm** — equivalents on the other clouds; both support native locking.

Worth mentioning: migrating between backends is `terraform init -migrate-state`, and you
always back up state before doing it.

### What are Terraform workspaces?

Multiple named state files within a single backend and configuration. Every configuration
starts in the `default` workspace.

```bash
terraform workspace new qa
terraform workspace list
terraform workspace select prod
terraform workspace show
```

Reference the current one as `terraform.workspace`:

```hcl
locals {
  instance_count = terraform.workspace == "prod" ? 3 : 1
}
```

With the S3 backend, non-default workspaces are stored under
`env:/<workspace>/<key>` (or the `workspace_key_prefix`).

### When should you use workspaces?

When the environments are **structurally identical** and differ only in scale/naming:

- Short-lived ephemeral environments — per-PR or per-feature-branch stacks.
- Developer sandboxes from the same config.
- Regional copies of an identical stack.
- Module testing.

### When should you avoid workspaces?

For **dev/staging/prod** in a serious setup. Reasons:

- Same backend and same credentials means a mis-selected workspace can apply to prod —
  there's no IAM boundary between them.
- Same configuration means environments can't diverge (prod needs Multi-AZ RDS, WAF, and
  a different account; dev doesn't).
- `terraform.workspace` conditionals accumulate into unreadable ternary soup.
- No per-environment provider/backend/version differences.
- The current workspace is invisible in code review — a PR diff doesn't tell you which
  state it will hit.

The recommended alternative: separate root directories (`live/dev`, `live/prod`) with
separate state keys, separate AWS accounts, and separate CI credentials. Explicit beats
implicit when the failure mode is "destroyed production."

---

## 8. Advanced

### What is terraform import?

Brings an existing, unmanaged resource under Terraform management by writing it into
state. The classic CLI form:

```bash
terraform import aws_ecr_repository.api stitched-api
```

Terraform 1.5+ added the declarative, plannable `import` block — reviewable in a PR and
runnable in CI:

```hcl
import {
  to = aws_ecr_repository.api
  id = "stitched-api"
}
```

Important: import only populates state — you still have to write matching configuration.
Terraform 1.5+ can generate a starting point with
`terraform plan -generate-config-out=generated.tf`. Then iterate until plan shows no
changes.

### How do you manage existing AWS infrastructure with Terraform?

1. **Inventory** what exists (Config, Resource Explorer, tags, `aws` CLI).
2. **Prioritize** — start with stable, low-risk, high-value resources (VPC, IAM, ECR),
   leave churn-heavy things for later.
3. **Write config or generate it** — `import` blocks +
   `-generate-config-out`, or tools like Terraformer / former2 for bulk scaffolding.
4. **Import incrementally**, one logical group per PR, into its own state.
5. **Iterate until `plan` is clean** — a no-op plan is the acceptance criterion. Any
   proposed destroy means your config doesn't match reality yet.
6. **Lock the door** — remove human write access via IAM so new drift can't appear, add
   `prevent_destroy` on the scary resources.
7. **Backfill tags** like `ManagedBy = terraform` so ownership is visible in the console.

### What is terraform taint?

The old command that marked a resource as degraded in state so the next apply would
destroy and recreate it (`terraform taint aws_instance.web`). It mutated state
immediately with no plan preview, which was its main flaw. **Deprecated in 0.15.2 and
removed in later versions.**

### What replaced terraform taint in newer versions?

The `-replace` flag on plan/apply, which is plan-visible and doesn't mutate state up front:

```bash
terraform plan  -replace="aws_instance.web" -out=tfplan
terraform apply -replace="aws_instance.web"
```

And declaratively, `replace_triggered_by` in a `lifecycle` block for
"rebuild Y when X changes."

### What is terraform console?

An interactive REPL that evaluates expressions against your current configuration and
state — the fastest way to debug HCL without running an apply.

```bash
$ terraform console
> var.environment
"prod"
> cidrsubnet("10.20.0.0/16", 4, 2)
"10.20.32.0/20"
> [for s in module.vpc.private_subnets : s]
> keys(var.services)
> aws_ecs_service.api.desired_count
```

Read-only, so it's safe. Invaluable for getting `for` expressions, `merge()`, and
`cidrsubnet()` math right on the first apply.

### What is the Terraform dependency graph?

A directed acyclic graph (DAG) of resources, data sources, modules, providers, and
variables, built from value references and `depends_on`. Terraform walks it to determine
what can run in parallel (default 10 concurrent operations, tunable with
`-parallelism=n`) and reverses it for destroy ordering. Cycles are a hard error
("Cycle: ..."), usually caused by two resources referencing each other's attributes.
Inspect it with:

```bash
terraform graph | dot -Tsvg > graph.svg
```

### What happens when Terraform detects drift?

Drift = real infrastructure diverged from state (someone changed it in the console, or
another system did). At the start of plan, Terraform refreshes and compares:

- If the config still specifies the original value, the plan shows a change that will
  **revert** the manual edit — Terraform reasserts the code as truth.
- Terraform 1.x reports it separately under "Note: Objects have changed outside of
  Terraform," which is your audit signal.
- If the resource was deleted out of band, plan proposes recreating it.
- Attributes under `ignore_changes` are excluded.
- Drift is only recorded in state when you apply (or run `-refresh-only`).

### How do you detect infrastructure drift?

- **Scheduled plan in CI** — a nightly/hourly GitHub Actions job running
  `terraform plan -detailed-exitcode`; exit code `2` means changes pending → alert Slack.
  This is the highest-value, lowest-effort control.
- **`terraform plan -refresh-only`** to see drift without config-driven noise.
- **AWS Config rules / conformance packs** for account-level policy drift.
- **CloudTrail alerts** on mutating API calls by non-Terraform principals.
- **HCP Terraform drift detection** if you're on that platform.
- **Prevention beats detection** — remove human write access in prod; make the pipeline
  the only path.

### What is the difference between terraform refresh and terraform plan?

`refresh` (now `apply -refresh-only`) syncs **state ← reality** and writes the updated
state; it never looks at whether your config wants something different. `plan` performs
that refresh in memory and then diffs **config vs state**, producing a proposed change
set without writing state. In short: refresh updates the record, plan proposes the work.

### What are Terraform functions?

Built-in functions for transforming values inside expressions. Terraform has no
user-defined functions in the classic sense (1.8+ added provider-defined functions), so
you compose the built-ins. They're pure and evaluated at plan time.

### Name some commonly used Terraform functions.

**String:** `format`, `join`, `split`, `replace`, `lower`/`upper`, `trimspace`,
`substr`, `startswith`, `regex`, `templatestring`

**Collection:** `length`, `keys`, `values`, `merge`, `concat`, `lookup`, `element`,
`contains`, `flatten`, `distinct`, `zipmap`, `toset`, `coalesce`, `try`, `one`

**Numeric:** `min`, `max`, `ceil`, `floor`, `abs`

**Encoding:** `jsonencode`, `jsondecode`, `yamlencode`, `base64encode`, `base64decode`

**Filesystem:** `file`, `templatefile`, `fileset`, `pathexpand`

**Network:** `cidrsubnet`, `cidrhost`, `cidrnetmask`

**Hash/crypto:** `sha256`, `md5`, `uuid`, `bcrypt`

**Type/date:** `tostring`, `tonumber`, `can`, `timestamp`, `timeadd`, `formatdate`

The ones that show up constantly in real AWS code:

```hcl
tags       = merge(local.common_tags, { Name = "${local.name_prefix}-api" })
subnet     = cidrsubnet(var.vpc_cidr, 4, count.index)
policy     = jsonencode({ Version = "2012-10-17", Statement = [...] })
user_data  = templatefile("${path.module}/init.sh.tftpl", { env = var.environment })
az_name    = element(data.aws_availability_zones.available.names, count.index)
value      = try(var.overrides[each.key], var.defaults)
```

### What are dynamic blocks?

(Also asked earlier — the short version.) A construct that generates repeated **nested
blocks** inside a resource from a collection, using `for_each` and a `content` block.
`count`/`for_each` multiply whole resources; `dynamic` multiplies blocks *within* one
resource. Iterator variable defaults to the block label and can be renamed with
`iterator = x`. They can't be used for top-level meta-arguments like `lifecycle` or
`provider`.

---

## 9. AWS-Specific

> These are asked as "walk me through it" questions. Keep the snippet minimal and spend
> your words on the decisions: encryption, subnet placement, IAM least privilege,
> lifecycle protection.

### How do you create an EC2 instance using Terraform?

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "app" {
  ami                    = data.aws_ami.al2023.id
  instance_type          = var.instance_type
  subnet_id              = module.vpc.private_subnets[0]
  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile   = aws_iam_instance_profile.app.name
  user_data              = templatefile("${path.module}/user_data.sh.tftpl", {})
  monitoring             = true

  metadata_options {
    http_tokens = "required" # IMDSv2 only
  }

  root_block_device {
    volume_type = "gp3"
    volume_size = 30
    encrypted   = true
  }

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-app" })
}
```

Talking points: AMI via data source not hardcoded ID, private subnet, instance profile
instead of static keys, IMDSv2 required, encrypted EBS, `user_data` rather than
provisioners.

### How do you provision a VPC using Terraform?

In production, use the community module rather than writing 40 resources by hand:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.8"

  name = "${local.name_prefix}-vpc"
  cidr = "10.20.0.0/16"

  azs             = slice(data.aws_availability_zones.available.names, 0, 3)
  public_subnets  = [for i in range(3) : cidrsubnet("10.20.0.0/16", 8, i)]
  private_subnets = [for i in range(3) : cidrsubnet("10.20.0.0/16", 8, i + 10)]

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "prod"  # cost vs HA
  enable_dns_hostnames   = true
  enable_flow_log        = true
  flow_log_destination_type = "cloud-watch-logs"

  tags = local.common_tags
}
```

Hand-rolled, the pieces are: `aws_vpc`, `aws_subnet` (public/private per AZ),
`aws_internet_gateway`, `aws_eip` + `aws_nat_gateway`, `aws_route_table` +
`aws_route_table_association`, and optionally `aws_vpc_endpoint` for S3/ECR (which also
cuts NAT data-transfer cost noticeably for ECS image pulls).

### How do you create an EKS cluster using Terraform?

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.24"

  cluster_name    = "${local.name_prefix}-eks"
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access       = true
  cluster_endpoint_public_access_cidrs = var.admin_cidrs
  cluster_encryption_config            = { resources = ["secrets"] }
  cluster_enabled_log_types            = ["api", "audit", "authenticator"]

  enable_irsa                              = true
  enable_cluster_creator_admin_permissions = true
  authentication_mode                      = "API_AND_CONFIG_MAP"

  eks_managed_node_groups = {
    default = {
      min_size       = 2
      max_size       = 6
      desired_size   = 3
      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
    }
  }

  tags = local.common_tags
}
```

Decisions to mention: private node subnets, IRSA (or Pod Identity) so pods get scoped IAM
roles instead of node-role permissions, control-plane logging to CloudWatch, secrets
envelope-encrypted with KMS, EKS access entries over raw `aws-auth`, and addons
(VPC CNI, CoreDNS, kube-proxy, EBS CSI) managed as `cluster_addons`. Cluster
infrastructure in Terraform; workloads in Helm/Argo CD, not in Terraform.

### How do you manage IAM roles using Terraform?

```hcl
data "aws_iam_policy_document" "assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ecs-tasks.amazonaws.com"]
    }
  }
}

data "aws_iam_policy_document" "task" {
  statement {
    effect    = "Allow"
    actions   = ["secretsmanager:GetSecretValue"]
    resources = [aws_secretsmanager_secret.db.arn]
  }
}

resource "aws_iam_role" "task" {
  name               = "${local.name_prefix}-task"
  assume_role_policy = data.aws_iam_policy_document.assume.json
  tags               = local.common_tags
}

resource "aws_iam_role_policy" "task" {
  role   = aws_iam_role.task.id
  policy = data.aws_iam_policy_document.task.json
}
```

Points to make: prefer `aws_iam_policy_document` over inline `jsonencode` for
readability and validation; scope `resources` to ARNs, never `"*"`; separate task role
from task execution role; use OIDC federation for CI rather than long-lived keys; and
validate policies with Checkov/`iam-policy-validator` in the pipeline.

### How do you create an ALB with Terraform?

```hcl
resource "aws_lb" "this" {
  name                       = "${local.name_prefix}-alb"
  load_balancer_type         = "application"
  subnets                    = module.vpc.public_subnets
  security_groups            = [aws_security_group.alb.id]
  enable_deletion_protection = var.environment == "prod"
  drop_invalid_header_fields = true

  access_logs {
    bucket  = aws_s3_bucket.logs.id
    prefix  = "alb"
    enabled = true
  }
}

resource "aws_lb_target_group" "api" {
  name        = "${local.name_prefix}-api-tg"
  port        = 3000
  protocol    = "HTTP"
  target_type = "ip"           # required for Fargate awsvpc
  vpc_id      = module.vpc.vpc_id

  health_check {
    path                = "/health"
    matcher             = "200"
    interval            = 15
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.this.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.this.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.this.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

### How do you create an Auto Scaling Group?

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "${local.name_prefix}-lt-"
  image_id      = data.aws_ami.al2023.id
  instance_type = "t3.medium"
  user_data     = base64encode(file("${path.module}/user_data.sh"))

  metadata_options { http_tokens = "required" }
  lifecycle { create_before_destroy = true }
}

resource "aws_autoscaling_group" "app" {
  name_prefix         = "${local.name_prefix}-asg-"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 3
  vpc_zone_identifier = module.vpc.private_subnets
  target_group_arns   = [aws_lb_target_group.api.arn]
  health_check_type   = "ELB"

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences { min_healthy_percentage = 90 }
  }

  tag {
    key                 = "Name"
    value               = "${local.name_prefix}-app"
    propagate_at_launch = true
  }

  lifecycle { ignore_changes = [desired_capacity] }
}

resource "aws_autoscaling_policy" "cpu" {
  name                   = "target-tracking-cpu"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60
  }
}
```

Key details: launch template (not the legacy launch configuration), `instance_refresh`
for rolling replacement, `ELB` health checks so unhealthy targets are replaced, and
`ignore_changes = [desired_capacity]` so Terraform doesn't fight the scaling policy.

### How do you manage Secrets Manager with Terraform?

```hcl
resource "aws_secretsmanager_secret" "db" {
  name                    = "${local.name_prefix}/database"
  kms_key_id              = aws_kms_key.secrets.arn
  recovery_window_in_days = 30
}

# Create the container in Terraform; let the value be set out-of-band
resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({ username = var.db_username, password = random_password.db.result })

  lifecycle { ignore_changes = [secret_string] } # rotation won't be reverted
}
```

The important interview point: **anything Terraform writes into a secret is stored in
plaintext in state.** So the good patterns are (a) create the secret resource in
Terraform but populate the value via console/CLI/rotation Lambda and `ignore_changes`,
or (b) generate it with `random_password` so no human ever sees it, keeping the state
backend encrypted and tightly IAM-scoped either way. Then consumers reference the ARN —
ECS `secrets` blocks pull it at task start, so the value never enters the task definition
or the pipeline logs.

### How do you create an S3 bucket with versioning?

Modern provider versions split this into separate resources:

```hcl
resource "aws_s3_bucket" "assets" {
  bucket = "${local.name_prefix}-assets"
  tags   = local.common_tags

  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "assets" {
  bucket                  = aws_s3_bucket.assets.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  rule {
    id     = "expire-noncurrent"
    status = "Enabled"
    noncurrent_version_expiration { noncurrent_days = 90 }
  }
}
```

Mention that inline `versioning {}` / `acl` arguments were removed in AWS provider v4+ —
a common upgrade gotcha.

### How do you manage Route 53 records?

```hcl
data "aws_route53_zone" "main" {
  name         = "example.com"
  private_zone = false
}

resource "aws_route53_record" "alb" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.this.dns_name
    zone_id                = aws_lb.this.zone_id
    evaluate_target_health = true
  }
}

# ACM DNS validation — the for_each idiom worth memorizing
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.this.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    }
  }

  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
  allow_overwrite = true
}
```

Alias records for AWS targets (free queries, no TTL to manage), plain `records` + `ttl`
for external targets, `aws_route53_health_check` plus weighted/failover routing policies
for HA.

### How do you provision RDS using Terraform?

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${local.name_prefix}-db"
  subnet_ids = module.vpc.database_subnets
}

resource "random_password" "db" {
  length  = 32
  special = false
}

resource "aws_db_instance" "main" {
  identifier     = "${local.name_prefix}-postgres"
  engine         = "postgres"
  engine_version = "16.3"
  instance_class = var.environment == "prod" ? "db.r6g.large" : "db.t4g.micro"

  allocated_storage     = 50
  max_allocated_storage = 200
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  db_name  = "app"
  username = "app_admin"
  password = random_password.db.result

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  multi_az               = var.environment == "prod"

  backup_retention_period    = var.environment == "prod" ? 30 : 7
  backup_window              = "03:00-04:00"
  maintenance_window         = "Mon:04:00-Mon:05:00"
  auto_minor_version_upgrade = false

  performance_insights_enabled    = true
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  monitoring_interval             = 60

  deletion_protection       = var.environment == "prod"
  skip_final_snapshot       = var.environment != "prod"
  final_snapshot_identifier = "${local.name_prefix}-final"
  apply_immediately         = false

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [password, engine_version]
  }

  tags = local.common_tags
}
```

Notes worth saying out loud: private subnets and `publicly_accessible = false`; encrypted
storage with a CMK; `random_password` so no credential is typed by a human; the generated
password pushed into Secrets Manager for the app to consume; `deletion_protection` +
`prevent_destroy` + `final_snapshot`; and `apply_immediately = false` so changes land in
the maintenance window instead of causing a surprise restart.

---

## 10. CI/CD Integration

### How do you integrate Terraform into GitHub Actions?

The standard two-phase workflow — plan on PR, apply on merge with an approval gate:

```yaml
name: terraform

on:
  pull_request:
    paths: ["infrastructure/**"]
  push:
    branches: [main]
    paths: ["infrastructure/**"]

permissions:
  id-token: write        # required for OIDC
  contents: read
  pull-requests: write   # to comment the plan

jobs:
  plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/live/prod
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/gha-terraform-plan
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.5

      - run: terraform fmt -check -recursive
      - run: terraform init -input=false
      - run: terraform validate
      - run: terraform plan -input=false -lock-timeout=5m -out=tfplan
      - run: terraform show -no-color tfplan > plan.txt

      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: infrastructure/live/prod/tfplan

  apply:
    if: github.ref == 'refs/heads/main'
    needs: plan
    runs-on: ubuntu-latest
    environment: production        # <- manual approval gate lives here
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: tfplan, path: infrastructure/live/prod }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/gha-terraform-apply
          aws-region: ap-south-1
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -input=false
      - run: terraform apply -input=false -auto-approve tfplan
```

Details that show experience: applying the **saved plan artifact** so you execute exactly
what was reviewed; separate read-only plan role and write apply role; `concurrency` groups
so two applies never race; posting the plan as a PR comment; adding `tflint`, `Checkov`,
and `Infracost` as PR checks.

### How do you secure AWS credentials in CI/CD?

**Use OIDC federation — no long-lived access keys at all.** GitHub's OIDC provider issues
a short-lived token, the runner exchanges it for temporary STS credentials, and the trust
policy restricts which repo/branch/environment can assume the role:

```hcl
data "aws_iam_policy_document" "gha_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:growthloops/infrastructure:ref:refs/heads/main"]
    }
  }
}
```

Also: pin the `sub` condition tightly (never `repo:org/*:*`), separate plan/apply roles,
scope secrets to GitHub **Environments** with required reviewers, never `echo` a secret,
add Gitleaks to pre-commit and CI, and keep state in an encrypted bucket with a scoped
bucket policy. Remember GitHub masks secrets in logs — which can silently blank a job
output containing an account ID, so pass SHAs and reconstruct ARNs rather than passing
ARNs between jobs.

### How do you prevent accidental production deployments?

Layered controls, because any single one can be bypassed:

- **Separate AWS accounts** per environment — the strongest boundary; dev credentials
  physically cannot reach prod.
- **Separate state files and root directories** — no workspace mis-selection risk.
- **GitHub Environment protection rules** — required reviewers, wait timers, and branch
  restrictions on the prod environment.
- **Prod applies only from `main`**, only via CI; branch protection + CODEOWNERS on
  `live/prod/**`.
- **No local prod credentials** — humans get read-only; the apply role is assumable only
  by the OIDC principal.
- **Saved plan artifacts** so apply can't drift from the reviewed plan.
- **`prevent_destroy` and `deletion_protection`** on stateful resources.
- **Policy as code** — OPA/Conftest or Sentinel failing plans that destroy databases or
  open `0.0.0.0/0`.
- **Never `-auto-approve`** except on a plan file that already passed review.

### How do you handle Terraform approvals in CI/CD?

Plan on PR → human reviews the diff in the PR comment → merge → the apply job pauses on a
protected GitHub Environment requiring reviewer sign-off → apply runs the stored plan.
Additions that matter: `-detailed-exitcode` to skip the approval entirely when there are
no changes; automated policy checks (Checkov, OPA, Infracost thresholds) that must pass
before a human is even asked; a `concurrency` group keyed to the environment so applies
serialize; and a Slack notification with a link to the pending approval. On HCP Terraform
the same shape exists natively with run tasks and Sentinel policies.

### How do you run Terraform in multiple environments?

Preferred: **directory-per-environment root modules** sharing versioned modules.

```
live/
├── dev/apps/{main.tf, backend.hcl, terraform.tfvars}
├── staging/apps/...
└── prod/apps/...
```

Each has its own backend key, its own AWS account/role, and its own tfvars. CI uses a
matrix or path filters to decide what to plan:

```yaml
strategy:
  matrix:
    env: [dev, staging, prod]
```

Alternatives to acknowledge: **workspaces** (good only for ephemeral/identical stacks),
**Terragrunt** (DRY backend/provider generation across many nearly-identical roots), and
**HCP Terraform workspaces** (one per env with variable sets). The non-negotiable in all
of them is separate state and separate credentials per environment, and promoting the
*same module version* dev → staging → prod so what you tested is what you ship.

---

## 11. Production Scenarios

> These are judgment questions. Interviewers want a calm, ordered process and evidence
> you've been on the wrong end of one.

### Your terraform apply fails halfway. What do you do?

Terraform has **no automatic rollback** — it applies as far as it got, and state reflects
what actually succeeded. So:

1. **Don't panic-rerun.** Read the error; identify which resource failed and why (IAM
   denial, quota, name conflict, dependency timing, API throttling).
2. **Check the lock** — if the run crashed, the lock may be stale; confirm nothing is
   running, then `force-unlock` with the ID from the error.
3. **Assess reality vs state** — `terraform plan` to see the remaining delta. Terraform
   is generally safe here: successful resources are in state, the failed one is not.
4. **Watch for the orphan case** — a resource created in the API but not recorded (crash
   between create and state write). That's the one you must find in the console and
   `import` or delete manually, or the next apply hits a name conflict.
5. **Fix forward.** Correct the config or the underlying cause and re-apply; Terraform
   converges on the desired state. Rolling back means reverting the commit and applying
   the previous version — which is a *new* apply, not an undo.
6. **Assess impact** — if production traffic is affected, mitigate first (scale the old
   service back up, revert DNS), then fix Terraform.
7. **Postmortem** — was it a missing IAM permission? Add it to the CI role. A quota? Raise
   it and add a preflight check. A partial-apply hazard? Split the stack smaller.

Also worth mentioning: `-target` can help you finish a specific piece in an emergency, but
it's an escape hatch that leaves state partially converged — always follow up with an
untargeted apply.

### Terraform wants to recreate a production resource. How would you prevent it?

1. **Stop.** Do not apply. Read the plan carefully to find *which attribute* forces
   replacement — Terraform prints `# forces replacement` next to it.
2. **Diagnose the cause**, because the fix depends on it:
   - Immutable attribute changed (RDS `engine`, subnet CIDR, EBS AZ) → revert the config
     to match reality, or plan a real migration with a snapshot/restore.
   - Provider upgrade changed a default → pin the provider, read the upgrade guide.
   - Resource was refactored/renamed → it's not a real change, it's an address change:
     use a `moved` block or `terraform state mv`.
   - Drift from a manual console edit → decide who's authoritative; `ignore_changes` if
     another system legitimately owns that field.
3. **Add the guardrail:** `lifecycle { prevent_destroy = true }` so any future plan that
   tries this fails loudly, plus `deletion_protection`/`skip_final_snapshot = false` on
   the AWS side.
4. If replacement is genuinely required, plan it as a **change with a runbook**:
   `create_before_destroy`, blue/green with DNS or target-group swap, snapshot taken
   first, maintenance window, rollback plan, and a reviewed plan file.

The meta-answer: this is why plans are reviewed by a human and why prod applies are
gated. `prevent_destroy` is what turns a 3am mistake into a failed pipeline.

### How do you manage separate dev, staging, and production environments?

- **Separate AWS accounts** under one Organization, with SCPs — the real isolation
  boundary for blast radius, quotas, billing, and IAM.
- **Separate root modules and state files** per environment; identical structure so
  promotion is boring.
- **Shared versioned modules** — the same module version moves dev → staging → prod, so
  what you tested is what ships.
- **Environment differences expressed as inputs**, not forked code: `multi_az`,
  `instance_class`, `desired_count`, `backup_retention_period`, log retention.
- **Separate CI credentials per environment** with an approval gate only on prod.
- **Prod-only extras** as explicit conditionals or a separate module: WAF, GuardDuty,
  cross-region backups, deletion protection.

Explicitly say why *not* workspaces for this: shared backend, shared credentials, config
that can't diverge, and the target environment isn't visible in the PR diff.

### How do you handle secrets in Terraform?

Core rule: **Terraform state stores everything in plaintext**, so the goal is to keep
secret *values* out of Terraform's hands entirely.

- **Reference, don't create** — Terraform manages the Secrets Manager/SSM
  SecureString/Vault *container* and IAM access; the value is injected out-of-band or by a
  rotation Lambda, with `ignore_changes = [secret_string]`.
- **Generate, never type** — `random_password` for DB credentials so no human ever handles
  it; write it straight to Secrets Manager.
- **Let the runtime fetch it** — ECS task definitions reference the secret ARN in
  `secrets`, EKS pods use IRSA + External Secrets Operator, so the value never appears in
  a task definition, a tfvars file, or a pipeline log.
- **Never commit** `*.tfvars` with secrets — gitignore them, and run Gitleaks in CI.
- **CI passes secrets as `TF_VAR_*` env vars** from GitHub Environment secrets, never CLI
  args (which land in process lists and logs).
- **Mark variables and outputs `sensitive = true`** — log hygiene, and understand its limits.
- **Protect state like a secret** — SSE-KMS, versioning, TLS-only bucket policy, tightly
  scoped IAM, and audit reads via CloudTrail.
- **Prefer OIDC/IRSA/instance profiles** over any static credential.

A good extra detail: passing a secret via ECS RunTask `overrides` puts it in the
**CloudTrail event**, so use the task definition's `secrets` block instead.

### How do you review Terraform changes before deployment?

- **`terraform plan` posted as a PR comment**, so the diff is reviewed alongside the code
  that produced it.
- **Read the destroy/replace lines first** — the summary count (`3 to add, 1 to change,
  0 to destroy`) is the headline; any unexpected `-` or `-/+` blocks the merge.
- **Automated gates before human review:** `fmt -check`, `validate`, `tflint`,
  `checkov`/`tfsec`, `infracost diff` for cost deltas, and OPA/Conftest policies (no
  public S3, no `0.0.0.0/0` SSH, mandatory tags, no unencrypted volumes).
- **CODEOWNERS** requiring a platform reviewer on `live/prod/**` and `modules/**`.
- **Small PRs** — one logical change per PR; a 400-resource plan gets rubber-stamped.
- **Apply the saved plan file**, so approval and execution can't diverge.
- **Test module changes in dev/staging first**; consider Terratest or `terraform test`
  for module contracts.

### How do you collaborate when multiple engineers work on the same infrastructure?

- **Remote state with locking** — non-negotiable; nobody keeps state locally.
- **CI is the only thing that applies.** Humans get read-only in prod. This removes the
  "who ran what from their laptop" class of incident entirely.
- **Split state by domain** — network / data / apps per environment, so two people working
  on unrelated things don't queue behind each other or share blast radius.
- **PR workflow with plan output and CODEOWNERS**; branch protection on `main`.
- **Version and changelog shared modules** so a module change doesn't surprise consumers.
- **`concurrency` groups** in CI keyed by state key, so applies serialize per stack.
- **Pin Terraform and provider versions** (`required_version`, `.terraform.lock.hcl`
  committed) so everyone's plan is identical.
- **Conventions written down** — naming, tagging, file layout — and enforced by lint
  rather than review comments.
- **Comms** — a Slack channel where the pipeline announces prod applies.

### How do you structure a large Terraform project?

```
infrastructure/
├── modules/                     # versioned, reusable, environment-agnostic
│   ├── vpc/
│   ├── ecs-service/
│   ├── rds/
│   └── observability/
├── live/
│   ├── dev/
│   │   ├── 00-network/          # rarely changes
│   │   ├── 10-data/             # RDS, ElastiCache — slow, stateful
│   │   ├── 20-platform/         # ECS cluster, ALB, ECR
│   │   └── 30-apps/             # services — changes daily
│   ├── staging/
│   └── prod/
├── global/
│   ├── iam/                     # OIDC providers, cross-account roles
│   └── dns/                     # Route 53 zones
├── policies/                    # OPA/Conftest rules
└── .github/workflows/
```

Guiding principles:

- **Split state by rate of change and blast radius.** Don't let an app deploy plan touch
  the VPC.
- **Directory per environment**, not workspaces.
- **Modules hold logic; roots hold composition and values.**
- **Cross-stack references are explicit** — remote state data sources, SSM parameters, or
  tag-based lookups. Prefer SSM/tags for loose coupling.
- **Keep plans fast** — a plan over 500 resources is a smell; split it.
- **Consistent naming/tagging via a `locals` convention** enforced by
  `default_tags` on the provider.
- **Terragrunt** if the backend/provider boilerplate across many roots becomes the pain
  point.

### How do you upgrade Terraform versions safely?

1. **Read the upgrade guide and changelog** for every version you're skipping. Never jump
   multiple minors blind, and note that state schema upgrades are one-way.
2. **Back up state** — confirm S3 versioning, and take an explicit `terraform state pull`
   snapshot.
3. **Bump `required_version`** with a constraint (`~> 1.9`) and use `tenv`/`tfenv` so
   local and CI versions match exactly.
4. **Upgrade one environment at a time**: sandbox → dev → staging → prod, letting each
   soak.
5. **Verify with a plan** — the acceptance test is `No changes.` If the new version
   proposes changes you didn't write, stop and investigate.
6. **Upgrade providers separately from Terraform itself** — `terraform init -upgrade`,
   review `.terraform.lock.hcl`, commit the lock file. Provider majors (like AWS v3→v4
   splitting S3 resources) cause far more churn than core upgrades.
7. **Coordinate with the team** — everyone updates at once, since a newer Terraform writes
   state that older versions can't read.
8. Keep the previous version installed so you can pin back quickly if CI breaks.

### What Terraform best practices do you follow?

**State:** remote backend, versioned + encrypted, locking on, one state per
environment/domain, never in Git.

**Code:** pin Terraform and provider versions, commit the lock file, `fmt` and `validate`
in CI, modules for anything used twice, `for_each` over `count`, descriptions and
validation on every variable, no hardcoded account IDs or ARNs, `default_tags` on the
provider.

**Security:** OIDC over static keys, least-privilege IAM scoped to ARNs, secrets in
Secrets Manager and referenced by ARN, `sensitive = true` where relevant, Checkov/tfsec
and Gitleaks in the pipeline, encryption on by default everywhere.

**Process:** plan on PR and apply the saved plan, mandatory review with CODEOWNERS,
approval gate on prod, separate accounts per environment, `prevent_destroy` +
`deletion_protection` on stateful resources, scheduled drift detection, small PRs, no
manual console changes.

**Operational:** `-lock-timeout` in CI, `concurrency` groups, cost visibility via
Infracost, and documented runbooks for lock recovery, state restore, and partial-apply
cleanup.

### What are the limitations of Terraform?

Honest answers here score well:

- **No automatic rollback** — a failed apply leaves partial state; you fix forward.
- **State is a liability** — it can be corrupted, lost, or leaked, and it holds secrets in
  plaintext. Managing it is real operational work.
- **Provider lag and bugs** — brand-new AWS features often aren't in the provider yet, and
  provider majors can force large refactors.
- **Not a configuration manager** — it provisions; it doesn't maintain in-place software
  state on running hosts.
- **Weak imperative/orchestration story** — no real loops beyond `count`/`for_each`, no
  conditional blocks, no try/retry logic; complex logic gets ugly fast.
- **Plan is not a guarantee** — it can succeed and the apply still fail on API-side
  validation, quotas, or race conditions.
- **Slow plans at scale** — refresh cost grows with resource count; forces you to split
  state.
- **Secrets handling requires discipline** — nothing in the tool protects you by default.
- **Drift is detected, not prevented** — you still need IAM controls to stop manual changes.
- **Refactoring is awkward** — renaming things means `moved` blocks or state surgery.
- **Destroy ordering and dependency edge cases** — hidden dependencies need `depends_on`;
  cycles need manual untangling.
- **Testing is immature** compared to application code, though `terraform test` and
  Terratest help.
- **Licensing** — Terraform moved to the BUSL in 2023, which is why OpenTofu exists; worth
  knowing as context if asked.

---

## Quick Command Reference

```bash
terraform init -upgrade -backend-config=backend.hcl
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan -detailed-exitcode      # 0=no change, 1=error, 2=changes
terraform apply tfplan
terraform apply -refresh-only                      # reconcile drift into state
terraform apply -replace="aws_instance.web"        # replaces old `taint`
terraform destroy -target=module.qa

terraform state list
terraform state show aws_ecs_service.api
terraform state mv module.old module.new
terraform state rm aws_s3_bucket.legacy
terraform state pull > backup.tfstate
terraform force-unlock <LOCK_ID>

terraform output -json
terraform output -raw alb_dns_name
terraform console
terraform graph | dot -Tsvg > graph.svg
terraform workspace list && terraform workspace select prod
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions[0]=="delete")'

TF_LOG=DEBUG terraform apply      # trace provider API calls
```

## Highest-Leverage Things to Say in the Interview

1. **State is the whole game** — remote, versioned, encrypted, locked, split by
   environment. Most Terraform incidents are state incidents.
2. **No rollback, so gate the apply** — plan on PR, apply the saved plan file, human
   approval on prod.
3. **`prevent_destroy` + `deletion_protection` + separate accounts** — layered guardrails,
   because one is never enough.
4. **OIDC, not access keys** — and separate read-only plan and write apply roles.
5. **`for_each` over `count`** and know why (reindexing destroys resources).
6. **Secrets never live in Terraform** — manage the container, not the value.
7. **`ignore_changes` is how Terraform and a deploy pipeline coexist** on the same ECS
   service.
8. Always be ready with **one real story**: something that broke, how you diagnosed it,
   what guardrail you added afterward.
