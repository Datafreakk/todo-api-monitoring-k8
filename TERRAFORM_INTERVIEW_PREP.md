# Terraform Interview Prep Notes

---

## TABLE OF CONTENTS

1. [What is Terraform & IaC](#1-what-is-terraform--iac)
2. [Core Concepts: Providers, Resources, Variables, Outputs](#2-core-concepts)
3. [Terraform State](#3-terraform-state)
4. [Modules](#4-modules)
5. [Loops: count, for_each, for expressions](#5-loops)
6. [Conditionals](#6-conditionals)
7. [Locals and Data Sources](#7-locals-and-data-sources)
8. [Functions](#8-functions)
9. [Lifecycle Rules](#9-lifecycle-rules)
10. [Workspaces](#10-workspaces)
11. [Remote Backends](#11-remote-backends)
12. [Terraform Commands Cheat Sheet](#12-terraform-commands-cheat-sheet)
13. [Terraform Workflow](#13-terraform-workflow)
14. [Common Interview Questions & Answers](#14-common-interview-questions--answers)

---

## 1. What is Terraform & IaC

### What is Infrastructure as Code (IaC)?
IaC means you describe your infrastructure (servers, networks, databases) in code files instead of clicking through a UI. This makes infrastructure reproducible, version-controlled, and automatable — just like application code.

### What is Terraform?
Terraform is an open-source IaC tool made by HashiCorp. You write configuration files in **HCL (HashiCorp Configuration Language)**, and Terraform figures out what needs to be created, changed, or destroyed to match your desired state.

**Key traits:**
- **Declarative** — you say *what* you want, not *how* to do it
- **Provider-agnostic** — works with AWS, Azure, GCP, Kubernetes, and hundreds more
- **Idempotent** — running `terraform apply` twice results in the same state, no duplicates
- **Execution plan** — shows you exactly what will change before doing it

### Terraform vs Ansible vs CloudFormation

| Tool | Type | Focus | Language |
|---|---|---|---|
| Terraform | Declarative IaC | Provisioning infra | HCL |
| Ansible | Procedural IaC / Config Mgmt | Config & app deployment | YAML |
| ARM Templates | Declarative IaC | Azure only | JSON/Bicep |

> **Interview tip:** Terraform is for provisioning infrastructure; Ansible is typically for configuring it after provisioning. They are often used together.

---

## 2. Core Concepts

### Providers
A provider is a plugin that lets Terraform talk to an external API (AWS, GCP, GitHub, etc.). You declare which provider you need and Terraform downloads it.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

- `source` — where to download the provider from (Terraform Registry)
- `version` — `~> 3.0` means any 3.x version, but not 4.x
- `features {}` — required empty block for the Azure provider (enables default feature flags)

### Resources
A resource is the primary building block. It describes a piece of infrastructure you want to create.

```hcl
resource "azurerm_resource_group" "main" {
  name     = "my-app-rg"
  location = "East US"
}

resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"

  tags = {
    Name = "WebServer"
  }
}
```

**Syntax:** `resource "<provider>_<type>" "<local_name>" { ... }`  
The local name (`web`) is used to reference this resource elsewhere.

### Variables (Input Variables)
Variables make your configuration reusable. You define them and pass values at runtime.

```hcl
# Define
variable "vm_size" {
  description = "Azure VM size"
  type        = string
  default     = "Standard_B1s"
}

# Use
resource "azurerm_linux_virtual_machine" "web" {
  size = var.vm_size
  # ... other required fields
}
```

**How to pass variable values:**
1. `terraform.tfvars` file (auto-loaded)
2. `-var="instance_type=t3.medium"` flag
3. Environment variable: `TF_VAR_instance_type=t3.medium`
4. Interactive prompt (if no default)

### Outputs
Outputs expose values from your configuration so you can see them after `apply` or use them in other modules.

```hcl
output "vm_public_ip" {
  description = "The public IP of the web server"
  value       = azurerm_linux_virtual_machine.web.public_ip_address
}
```

After `terraform apply`, outputs are printed to the terminal. You can also query with `terraform output vm_public_ip`.

---

## 3. Terraform State

### What is State?
State is how Terraform remembers what it has already created. It stores the mapping between your HCL configuration and the real-world resources. By default it's stored in a local file called `terraform.tfstate`.

Think of state as Terraform's memory — without it, Terraform wouldn't know what exists and would try to create everything from scratch every time.

### What does the state file contain?
- Resource IDs (e.g., the actual Azure resource ID `/subscriptions/abc-123.../resourceGroups/my-app-rg`)
- Attribute values (IP addresses, resource IDs, etc.)
- Dependency graph metadata
- Provider metadata

### Why is state important?
- Terraform uses state to build the **diff** (what needs to change)
- It tracks dependencies between resources
- It stores sensitive values like passwords (so protect it carefully)

### Remote State
In a team, storing state locally is dangerous — two people could apply at the same time and corrupt it. Remote backends solve this by storing state in a shared location with locking.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "my-terraform-rg"
    storage_account_name = "myterraformstate"
    container_name       = "tfstate"
    key                  = "prod/vpc/terraform.tfstate"
  }
}
```

**Common remote backends:** Azure Blob Storage (native locking), S3 (+ DynamoDB for locking), GCS, Terraform Cloud

### State Locking
When someone runs `terraform apply`, the state is **locked** so no one else can run apply at the same time. Azure Blob Storage uses native lease-based locking — no extra service needed.

### Useful State Commands

| Command | What it does |
|---|---|
| `terraform state list` | Lists all resources in state |
| `terraform state show azurerm_resource_group.main` | Shows details of one resource |
| `terraform state mv` | Renames/moves a resource in state (without destroying it) |
| `terraform state rm` | Removes a resource from state (Terraform forgets it, doesn't destroy) |
| `terraform import` | Imports an existing real resource into state |

### terraform.tfstate vs terraform.tfstate.backup
Every time state is updated, the previous version is saved as `terraform.tfstate.backup`. Useful for manual recovery.

> **Interview tip:** "Never manually edit the state file" is a best practice. Use `terraform state` commands instead.

---

## 4. Modules

### What is a Module?
A module is just a folder containing `.tf` files. Every Terraform project is technically a module — the root module. You create child modules to group related resources and reuse them.

Think of a module like a function in code — you define it once, call it many times with different inputs.

### Why use Modules?
- **Reusability** — define a VNet module once, use it in dev/staging/prod
- **Encapsulation** — hide implementation details, expose only necessary variables
- **Organization** — keeps large codebases manageable

### Module Structure Example

```
modules/
  vnet/
    main.tf       # resource definitions
    variables.tf  # input variables
    outputs.tf    # values exported from module
```

**Defining the module (modules/vnet/main.tf):**
```hcl
resource "azurerm_virtual_network" "main" {
  name                = var.name
  address_space       = [var.address_space]
  location            = var.location
  resource_group_name = var.resource_group_name
}
```

**Calling the module (root main.tf):**
```hcl
module "production_vnet" {
  source              = "./modules/vnet"
  name                = "production-vnet"
  address_space       = "10.0.0.0/16"
  location            = "East US"
  resource_group_name = "my-app-rg"
}

# Access module output
output "vnet_id" {
  value = module.production_vnet.vnet_id
}
```

### Module Sources
Modules can come from different places:

| Source | Example |
|---|---|
| Local path | `source = "./modules/vnet"` |
| Terraform Registry | `source = "Azure/network/azurerm"` |
| GitHub | `source = "github.com/org/repo//modules/vnet"` |
| Azure Blob / GCS | `source = "azurerm::https://..."` |

### Public Registry Example
```hcl
module "network" {
  source  = "Azure/network/azurerm"
  version = "5.3.0"

  resource_group_name = "my-app-rg"
  vnet_name           = "my-vnet"
  address_space       = "10.0.0.0/16"
}
```

> **Interview tip:** You need to run `terraform init` after adding or changing a module source — it downloads the module.

---

## 5. Loops

### Why loops?
If you need 3 Azure VMs or 5 storage accounts, you don't want to copy-paste the resource block 3 or 5 times. Loops let you create multiple resources from a single definition.

---

### `count` — simple integer-based repetition

```hcl
resource "azurerm_resource_group" "web" {
  count    = 3
  name     = "web-rg-${count.index}"   # web-rg-0, web-rg-1, web-rg-2
  location = "East US"
}
```


- `count.index` gives you the current iteration number (starts at 0)
- Resources are addressed as `azurerm_resource_group.web[0]`, `azurerm_resource_group.web[1]`, etc.

**Problem with count:** Resources are identified by index. If you remove item at index 0, Terraform renumbers everyone — causing destructive updates on resources you didn't intend to touch.

---

### `for_each` — map or set-based iteration

`for_each` is the preferred way to create multiple resources. Resources are identified by a unique key, not an index.

```hcl
variable "environments" {
  default = ["dev", "staging", "prod"]
}

resource "azurerm_resource_group" "env_rgs" {
  for_each = toset(var.environments)
  name     = "myapp-${each.key}-rg"
  location = "East US"
}
```

- `each.key` — the current key (for a set, key == value)
- `each.value` — the current value (for a map, this is the object)
- Resources addressed as `azurerm_resource_group.env_rgs["dev"]`, etc.

**With a map:**
```hcl
variable "container_apps" {
  default = {
    api     = { image = "myregistry.azurecr.io/todo-api:latest",   cpu = 0.5, memory = "1Gi" }
    worker  = { image = "myregistry.azurecr.io/todo-worker:latest", cpu = 0.25, memory = "0.5Gi" }
    gateway = { image = "myregistry.azurecr.io/gateway:latest",     cpu = 0.5, memory = "1Gi" }
  }
}

resource "azurerm_container_app" "apps" {
  for_each = var.container_apps

  name                         = each.key                          # "api", "worker", "gateway"
  resource_group_name          = azurerm_resource_group.main.name
  container_app_environment_id = azurerm_container_app_environment.env.id
  revision_mode                = "Single"

  template {
    container {
      name   = each.key
      image  = each.value.image     # from the map value object
      cpu    = each.value.cpu
      memory = each.value.memory
    }
  }
}
```

> **count vs for_each:** Use `count` for simple "create N identical things". Use `for_each` when items have distinct identities (names, configs) — deletion/addition won't disturb other items.

---

### `for` expressions — transforming collections

`for` expressions are used inside expressions to transform lists and maps. They don't create resources — they transform data.

```hcl
# Transform a list of strings to uppercase
variable "names" {
  default = ["alice", "bob", "carol"]
}

locals {
  upper_names = [for name in var.names : upper(name)]
  # Result: ["ALICE", "BOB", "CAROL"]
}
```

**Producing a map:**
```hcl
locals {
  name_lengths = { for name in var.names : name => length(name) }
  # Result: { alice = 5, bob = 3, carol = 5 }
}
```

**With a filter:**
```hcl
locals {
  long_names = [for name in var.names : name if length(name) > 3]
  # Result: ["alice", "carol"]
}
```

---

## 6. Conditionals

### Ternary expression
Terraform uses the classic ternary `condition ? true_value : false_value`.

```hcl
locals {
  environments = ["prod", "nonprod", "staging"]
}

resource "azurerm_service_plan" "this" {
  for_each = toset(local.environments)

  name                = "asp-${each.key}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  os_type             = "Linux"

  # ternary: condition ? value_if_true : value_if_false
  sku_name       = each.key == "prod" ? "P1v3" : "B2"
  instance_count = each.key == "prod" ? 3 : 1
  zone_redundant = each.key == "prod" ? true : false
}
```

### Conditional resource creation with `count`
```hcl
variable "create_dns_record" {
  type    = bool
  default = false
}

resource "azurerm_dns_a_record" "app" {
  count = var.create_dns_record ? 1 : 0
  # ... resource config
}
```

If `create_dns_record` is false, `count = 0` and the resource is not created at all. If true, `count = 1` and it's created once.

---

## 7. Locals and Data Sources

### Locals
Locals let you define intermediate values to avoid repetition. They're like local variables inside a function.

```hcl
locals {
  app_name    = "myapp"
  environment = "prod"
  common_tags = {
    App         = local.app_name
    Environment = local.environment
    ManagedBy   = "terraform"
  }
}

resource "azurerm_resource_group" "main" {
  name     = "my-app-${local.environment}-rg"
  location = "East US"
  tags     = local.common_tags   # reuse everywhere
}
```

### Data Sources
Data sources let you **read** existing infrastructure that wasn't created by your Terraform config. You use them to look up IDs, existing resource groups, or other values from Azure.

```hcl
# Look up an existing resource group to deploy into
data "azurerm_resource_group" "existing" {
  name = "my-existing-rg"
}

# Use it in a resource
resource "azurerm_linux_virtual_machine" "web" {
  resource_group_name = data.azurerm_resource_group.existing.name
  location            = data.azurerm_resource_group.existing.location
  # ...
}
```

Another common use: look up an existing Virtual Network.
```hcl
data "azurerm_virtual_network" "existing" {
  name                = "production-vnet"
  resource_group_name = "my-networking-rg"
}
```

> **Key difference:** `resource` creates/manages infrastructure. `data` only reads existing infrastructure.

---

## 8. Functions

Terraform has built-in functions for transforming and combining values. You can't define custom functions — only use built-ins.

### String Functions
```hcl
upper("hello")           # "HELLO"
lower("HELLO")           # "hello"
trimspace("  hi  ")      # "hi"
replace("foo-bar", "-", "_")  # "foo_bar"
format("Hello, %s!", "world") # "Hello, world!"
```

### Collection Functions
```hcl
length(["a", "b", "c"])       # 3
concat(["a"], ["b", "c"])     # ["a", "b", "c"]
flatten([["a", "b"], ["c"]])  # ["a", "b", "c"]
merge({a=1}, {b=2})           # {a=1, b=2}
keys({a=1, b=2})              # ["a", "b"]
values({a=1, b=2})            # [1, 2]
contains(["a","b"], "a")      # true
```

### Numeric Functions
```hcl
max(5, 12, 9)    # 12
min(5, 12, 9)    # 5
abs(-5)          # 5
ceil(1.2)        # 2
floor(1.9)       # 1
```

### Type Conversion
```hcl
tostring(42)         # "42"
tonumber("42")       # 42
tolist(toset([1,2])) # [1, 2]
tomap({a = "1"})     # {a = "1"}
```

### `lookup` — safe map access
```hcl
lookup({a = "val_a", b = "val_b"}, "a", "default")  # "val_a"
lookup({a = "val_a"}, "c", "default")               # "default"
```

> **Interview tip:** `terraform console` opens an interactive REPL where you can test functions directly. Very useful for debugging expressions.

---

## 9. Lifecycle Rules

### What are Lifecycle Rules?
Lifecycle rules give you control over how Terraform handles resource creation, updates, and deletion. You put them inside a `lifecycle` block within a resource.

### `create_before_destroy`
By default, Terraform destroys the old resource first, then creates the new one. This causes downtime. `create_before_destroy` reverses this.

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = "my-app-rg"
  location            = "East US"
  size                = "Standard_B1s"

  lifecycle {
    create_before_destroy = true
  }
}
```

Use this for stateless resources like Azure VMs behind a load balancer.

### `prevent_destroy`
Protects critical resources from accidental deletion. Terraform will error if you try to destroy or make a change that would require replacing this resource.

```hcl
resource "azurerm_mssql_server" "production_db" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

> This only protects against `terraform destroy` or plans that would destroy it. It does NOT protect against `terraform state rm` followed by `destroy`.

### `ignore_changes`
Tells Terraform to ignore changes to specific attributes made outside of Terraform (e.g., by autoscaling or manual edits).

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "app" {
  instances = 3
  # ...
  lifecycle {
    ignore_changes = [instances]  # autoscaling manages this
  }
}
```

You can use `ignore_changes = all` to ignore all attribute changes except those explicitly declared.

---

## 10. Workspaces

### What are Workspaces?
Workspaces let you manage multiple separate state files using the same configuration. Each workspace has its own independent state.

Think of workspaces like Git branches — same code, different environments.

```bash
terraform workspace new dev       # create a new workspace
terraform workspace new prod
terraform workspace select dev    # switch to dev workspace
terraform workspace list          # show all workspaces
terraform workspace show          # show current workspace
```

### Using workspace in config
```hcl
resource "azurerm_linux_virtual_machine" "web" {
  size = terraform.workspace == "prod" ? "Standard_D4s_v3" : "Standard_B1s"

  tags = {
    Environment = terraform.workspace
  }
}
```

### Workspaces vs separate directories
| | Workspaces | Separate directories |
|---|---|---|
| Code | Same | Duplicated or separate |
| State | Separate per workspace | Separate |
| Config differences | Via `terraform.workspace` conditionals | Different variable files |
| Recommended for | Small differences between envs | Large, complex differences |

> **Interview tip:** HashiCorp recommends using separate directories/repos for different environments in production. Workspaces work best for simple use cases.

---

## 11. Remote Backends

### What is a Backend?
A backend is where Terraform stores its state and (optionally) runs operations. The default backend is `local` (state in a local file).

### Why use a Remote Backend?
1. **Team collaboration** — shared state, everyone sees the same version
2. **State locking** — prevents concurrent applies
3. **Security** — state can contain secrets; storing in encrypted Azure Blob Storage is safer than a laptop
4. **CI/CD** — pipelines need shared state

### Azure Blob Storage Backend (most common for Azure)
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "my-company-terraform-rg"
    storage_account_name = "mycompanytfstate"
    container_name       = "tfstate"
    key                  = "services/todo-api/terraform.tfstate"
  }
}
```

**Setup required:**
1. Create the Storage Account manually (can't use Terraform to create its own backend)
2. Create a blob container named `tfstate` inside the storage account
3. No separate locking service needed — Azure Blob uses native lease-based locking

### Terraform Cloud Backend
```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "todo-api-prod"
    }
  }
}
```

### Partial Configuration (for sensitive values)
You can leave some backend config out of the `.tf` file and pass it at `init` time:
```bash
terraform init \
  -backend-config="storage_account_name=mycompanytfstate" \
  -backend-config="key=prod/terraform.tfstate"
```

---

## 12. Terraform Commands Cheat Sheet

| Command | What it does |
|---|---|
| `terraform init` | Downloads providers and modules, initializes backend |
| `terraform validate` | Checks syntax and configuration validity (no cloud API calls) |
| `terraform fmt` | Formats `.tf` files to canonical style |
| `terraform plan` | Shows what changes will be made (dry run) |
| `terraform apply` | Creates/updates/destroys infrastructure to match config |
| `terraform destroy` | Destroys all managed resources |
| `terraform output` | Shows output values |
| `terraform show` | Shows current state in a human-readable format |
| `terraform graph` | Outputs dependency graph (DOT format) |
| `terraform import` | Imports existing resource into state |
| `terraform taint` *(deprecated)* | Marks resource for replacement on next apply |
| `terraform replace` | Forces replacement of a specific resource |
| `terraform refresh` | Updates state to match real-world resources |
| `terraform console` | Interactive expression evaluator |

### Useful flags
```bash
terraform plan -out=plan.tfplan        # save plan to file
terraform apply plan.tfplan            # apply saved plan (no prompt)
terraform apply -auto-approve          # skip confirmation prompt (CI/CD)
terraform plan -target=azurerm_linux_virtual_machine.web  # plan only specific resource
terraform apply -var="env=prod"          # pass inline variable
terraform destroy -target=azurerm_resource_group.logs  # destroy one resource
```

---

## 13. Terraform Workflow

### Standard team workflow:

```
Write .tf files
       ↓
terraform init     — download providers/modules, set up backend
       ↓
terraform validate — check syntax
       ↓
terraform fmt      — format code (run in CI)
       ↓
terraform plan     — generate & review execution plan
       ↓
Code review of plan output
       ↓
terraform apply    — execute the plan
       ↓
terraform output   — view outputs
```

### What happens during `terraform apply`?
1. Terraform reads your `.tf` files
2. Reads current state from backend
3. Calls cloud provider APIs to refresh real-world state
4. Computes diff (what needs to change)
5. Asks for confirmation
6. Makes API calls to create/update/delete resources
7. Updates state file

### Resource Change Types
| Symbol | Meaning |
|---|---|
| `+` | Resource will be created |
| `-` | Resource will be destroyed |
| `~` | Resource will be updated in-place |
| `-/+` | Resource will be destroyed and re-created (replacement) |

---

## 14. Common Interview Questions & Answers

---

**Q: What is the difference between `terraform plan` and `terraform apply`?**

`plan` is a dry run — it shows you what changes Terraform would make without actually making them. `apply` executes those changes. In production, you typically run `plan`, review the output, then run `apply`. You can save a plan with `-out=plan.tfplan` and apply exactly that plan later.

---

**Q: What is Terraform state and why is it important?**

State is a JSON file that maps your HCL resources to real-world infrastructure (e.g., your `azurerm_linux_virtual_machine.web` maps to an Azure resource ID like `/subscriptions/.../resourceGroups/my-app-rg/providers/Microsoft.Compute/virtualMachines/web-vm`). Terraform uses it to detect drift (differences between config and reality) and compute what needs to change. Without state, Terraform would recreate everything on every run.

---

**Q: What is the difference between `count` and `for_each`?**

`count` creates N copies identified by index (0, 1, 2...). If you remove an item from the middle of a list, indexes shift and Terraform may destroy/recreate things you didn't want to change. `for_each` uses stable keys (names, IDs), so removing one item doesn't affect others. Prefer `for_each` for any meaningful resources.

---

**Q: How do you handle secrets in Terraform?**

Secrets should never be hardcoded in `.tf` files. Options:
- Pass via environment variables (`TF_VAR_db_password`)
- Use a secrets manager (Azure Key Vault, HashiCorp Vault) with a data source
- Mark sensitive variables with `sensitive = true` to hide them from plan output
- Use a remote backend with encryption for state (since state stores sensitive values in plaintext)

---

**Q: What is `terraform import` used for?**

It imports an existing real-world resource into Terraform's state without destroying or recreating it. Useful when you have manually-created infrastructure you want to bring under Terraform management. After import, you need to write the matching HCL configuration manually.

```bash
terraform import azurerm_resource_group.main /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/my-app-rg
```

---

**Q: What is the purpose of `terraform.tfvars`?**

It's a file where you set variable values. Terraform automatically loads `terraform.tfvars` and any file matching `*.auto.tfvars`. It lets you separate variable values from variable definitions — useful for having `dev.tfvars` and `prod.tfvars` for different environments.

---

**Q: What is a null resource and when would you use it?**

`null_resource` is a resource that does nothing by itself but lets you run `provisioners` (local-exec, remote-exec scripts). It's used when you need to run arbitrary scripts as part of your Terraform plan — for example, running a database migration script after an Azure SQL server is created.

---

**Q: What does `terraform refresh` do?**

It updates the state file to match the real-world state by querying the provider APIs. It doesn't make any changes to infrastructure. Note: `terraform plan` automatically does a refresh first, so `terraform refresh` alone is rarely needed.

---

**Q: What is a data source vs a resource?**

A resource creates and manages infrastructure (Terraform owns it — it can create, update, or delete it). A data source only reads/looks up existing infrastructure and makes the data available for use in your config. It never creates or modifies anything.

---

**Q: How do you prevent a Terraform resource from being accidentally deleted?**

Use the `prevent_destroy = true` lifecycle rule. This makes Terraform return an error if any plan would destroy that resource. It's commonly used on production databases and state storage accounts.

---

**Q: What happens if two people run `terraform apply` at the same time?**

If using a remote backend with locking (e.g., Azure Blob Storage, which uses native lease-based locking), the second person will get a lock error and their apply will fail — which is the correct behavior. If using local state, there's no locking and you could get state corruption, which is a major reason to use remote backends in a team.

---

**Q: What is a Terraform module and why use one?**

A module is a reusable package of Terraform configuration. You define a module once (e.g., a VNet with subnets and route tables) and call it multiple times with different inputs. This follows the DRY principle — don't repeat yourself — and makes large infrastructure codebases maintainable.

---

**Q: What is the `depends_on` meta-argument?**

Terraform automatically figures out dependencies by analyzing resource references. But sometimes there are implicit dependencies that Terraform can't detect. `depends_on` lets you explicitly declare that one resource depends on another.

```hcl
resource "azurerm_storage_blob" "config" {
  storage_account_name   = azurerm_storage_account.app.name
  storage_container_name = azurerm_storage_container.app.name
  # Terraform already knows to create the storage account first because of the reference above.
}

# But if the dependency isn't expressed via a reference:
resource "azurerm_role_assignment" "app" {
  depends_on = [azurerm_user_assigned_identity.app]  # explicit dependency
  # ...
}
```

---

**Q: What are provisioners and should you use them?**

Provisioners run scripts on local or remote machines during resource creation or destruction (`local-exec`, `remote-exec`). HashiCorp considers them a **last resort** because they make Terraform less idempotent and harder to test. Prefer using native provider features, user_data scripts, or configuration management tools like Ansible instead.

---

Alright, here's how you'd answer it naturally in an interview, start to finish:

"How do you manage Terraform module updates across environments?"

"So in our setup, modules live separately from the environment configurations. Each environment has its own folder with its own Terraform files, and that's where the module reference lives — including the version. So if I want to update a module, I bump the version ref in the dev environment first, raise a PR, that triggers the pipeline for dev only. Pipeline does init, plan, I review the plan output, approve, it applies. Once that looks good I do the same for staging, then prod. Each promotion is a separate PR and a separate pipeline run, so prod never gets touched until the change has been validated in lower environments first."


Then if they ask about the pipeline structure:

"We had one pipeline definition parameterised by environment rather than three separate files — less duplication. But the trigger was path-based, so a change in the dev folder only kicks off the dev pipeline run. Prod had an additional manual approval gate before apply, so even if something merged it wouldn't auto-deploy without a human signing off."


And if they push on why sequential:

"It's about change safety really. A module update could have unintended side effects — resource recreation, a drift in behaviour. You want to catch that in dev where the blast radius is low, not find out in prod. Sequential promotion with plan review at each stage is how you keep that control."

*Last updated: April 2026 | Covers Terraform ~1.7+*
