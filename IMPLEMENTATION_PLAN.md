# Kustomize-TF: Comprehensive Implementation Plan

## Executive Summary

This document presents a detailed technical plan for building **kustomize-tf**, a CLI tool that brings Kustomize's base + overlay pattern to Terraform. The tool will allow teams to define reusable base configurations and create environment-specific overlays that merge cleanly, eliminating code duplication while maintaining flexibility.

---

## Table of Contents

1. [Architecture Design](#1-architecture-design)
2. [Technical Decisions](#2-technical-decisions)
3. [Merge Semantics Deep Dive](#3-merge-semantics-deep-dive)
4. [CLI Interface Design](#4-cli-interface-design)
5. [Implementation Phases](#5-implementation-phases)
6. [Testing Strategy](#6-testing-strategy)
7. [Open Questions & Risks](#7-open-questions--risks)

---

## 1. Architecture Design

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              kustomize-tf CLI                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   Command   │───▶│   Config    │───▶│    Merge    │───▶│  Executor   │  │
│  │   Parser    │    │   Loader    │    │   Engine    │    │             │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                            │                  │                  │          │
│                            ▼                  ▼                  ▼          │
│                     ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│                     │     HCL     │    │    Temp     │    │  Terraform  │  │
│                     │   Parser    │    │   Writer    │    │   Process   │  │
│                     └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Core Components

#### Component 1: Command Parser
**Responsibility:** Parse CLI arguments and route to appropriate handlers

```go
type Command struct {
    Name      string           // plan, apply, build, destroy, init
    OverlayPath string         // path to overlay directory (-k flag)
    TFArgs    []string         // passthrough args to terraform
    DryRun    bool             // show merged config without executing
    Verbose   bool             // detailed logging
}
```

#### Component 2: Config Loader
**Responsibility:** Locate and load kustomize.tf, resolve base/overlay paths, validate structure

```go
type KustomizeConfig struct {
    Bases      []string          // paths to base directories (relative to overlay)
    Resources  []string          // additional .tf files to include
    Patches    []PatchConfig     // explicit patch files (Phase 2)
}

type LoadedConfig struct {
    OverlayPath  string
    BasePath     string
    BaseFiles    map[string]*hclwrite.File    // filename -> parsed HCL
    OverlayFiles map[string]*hclwrite.File    // filename -> parsed HCL
    Kustomize    KustomizeConfig
}
```

#### Component 3: HCL Parser
**Responsibility:** Parse .tf files into AST, preserving tokens for expressions

```go
type HCLParser struct {
    // Uses github.com/hashicorp/hcl/v2/hclwrite for AST manipulation
}

// Key insight: hclwrite preserves expressions as raw tokens
// This means var.foo, local.bar, function calls stay intact
func (p *HCLParser) ParseFile(path string) (*hclwrite.File, error)
func (p *HCLParser) ParseDir(dir string) (map[string]*hclwrite.File, error)
```

#### Component 4: Merge Engine
**Responsibility:** Implement the core merge logic between base and overlay

```go
type MergeEngine struct {
    Strategy MergeStrategy
}

type MergeStrategy interface {
    MergeBlocks(base, overlay *hclwrite.Block) *hclwrite.Block
    MergeAttributes(base, overlay *hclwrite.Body) *hclwrite.Body
    MergeMaps(base, overlay map[string]cty.Value) map[string]cty.Value
}

// The merged result is a new hclwrite.File that can be serialized
func (m *MergeEngine) Merge(base, overlay *hclwrite.File) (*hclwrite.File, error)
```

#### Component 5: Temp Writer
**Responsibility:** Write merged configuration to a temporary directory

```go
type TempWriter struct {
    TempDir string
}

// Creates isolated temp directory with merged .tf files
func (w *TempWriter) Write(merged map[string]*hclwrite.File) (string, error)
func (w *TempWriter) Cleanup() error
```

#### Component 6: Terraform Executor
**Responsibility:** Execute terraform commands against merged configuration

```go
type Executor struct {
    TerraformPath string  // path to terraform binary
    WorkDir       string  // temp directory with merged config
}

func (e *Executor) Init(args []string) error
func (e *Executor) Plan(args []string) error
func (e *Executor) Apply(args []string) error
func (e *Executor) Destroy(args []string) error
```

### 1.3 Data Flow

```
1. User runs: kustomize-tf plan -k overlays/prod

2. Command Parser:
   - Parse command (plan)
   - Extract overlay path (overlays/prod)
   - Collect terraform args

3. Config Loader:
   - Read overlays/prod/kustomize.tf
   - Resolve base path (e.g., ../../base)
   - List all .tf files in base and overlay

4. HCL Parser:
   - Parse all base .tf files into AST
   - Parse all overlay .tf files into AST
   - Preserve expressions as tokens (not evaluated)

5. Merge Engine:
   - For each block type (resource, variable, output, local, module):
     a. Match blocks by identifier (type+name for resources)
     b. Apply merge rules (scalars replace, maps deep merge)
     c. Add new blocks from overlay
   - Produce merged AST

6. Temp Writer:
   - Create temp directory
   - Serialize merged AST to .tf files
   - Copy any non-.tf files needed (e.g., .terraform.lock.hcl)

7. Executor:
   - cd to temp directory
   - Run: terraform plan [args]
   - Stream output to user
   - Cleanup temp directory on exit
```

### 1.4 Key Data Structures

```go
// Block identity for matching base to overlay
type BlockIdentity struct {
    Type   string   // "resource", "variable", "module", etc.
    Labels []string // ["aws_instance", "web"] for resources
}

func (b BlockIdentity) Key() string {
    return b.Type + "." + strings.Join(b.Labels, ".")
}

// Represents the merge result for a single block
type MergedBlock struct {
    Identity BlockIdentity
    Source   string          // "base", "overlay", "merged"
    Block    *hclwrite.Block
}

// Overall merge result
type MergeResult struct {
    Blocks   map[string]*MergedBlock  // keyed by BlockIdentity.Key()
    Warnings []string
    Errors   []error
}
```

---

## 2. Technical Decisions

### 2.1 Language Choice: **Go**

**Justification:**

| Factor | Go | Rust | Python |
|--------|-----|------|--------|
| HCL Library Quality | ★★★★★ Official hashicorp/hcl | ★★★☆☆ Community hcl-rs | ★★☆☆☆ python-hcl2 (limited) |
| Terraform Ecosystem | ★★★★★ Same language, same libs | ★★★☆☆ External | ★★☆☆☆ External |
| Single Binary Distribution | ★★★★★ Native | ★★★★★ Native | ★★☆☆☆ Requires runtime |
| Development Speed | ★★★★☆ Fast | ★★★☆☆ Slower | ★★★★★ Fastest |
| Performance | ★★★★☆ Excellent | ★★★★★ Best | ★★☆☆☆ Adequate |

**Decision:** Go is the clear choice because:
1. **hashicorp/hcl v2** is the authoritative HCL library, written in Go
2. **hclwrite** package provides exactly what we need: AST manipulation while preserving tokens
3. Single binary distribution matches terraform's model
4. Existing tools (Terragrunt, Atmos, Terramate) are all in Go

### 2.2 Essential Libraries

```go
import (
    // Core HCL processing
    "github.com/hashicorp/hcl/v2"
    "github.com/hashicorp/hcl/v2/hclparse"
    "github.com/hashicorp/hcl/v2/hclwrite"
    "github.com/hashicorp/hcl/v2/hclsyntax"

    // Type system for values
    "github.com/zclconf/go-cty/cty"
    "github.com/zclconf/go-cty/cty/json"

    // CLI framework
    "github.com/spf13/cobra"

    // File utilities
    "github.com/otiai10/copy"  // directory copying

    // Testing
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)
```

### 2.3 Why hclwrite is Critical

The `hclwrite` package solves the hardest problem: **preserving expressions without evaluation**.

```go
// Example: parsing and preserving expressions
content := []byte(`
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = local.instance_type
  tags          = merge(local.common_tags, { Name = "web" })
}
`)

file, _ := hclwrite.ParseConfig(content, "main.tf", hcl.Pos{Line: 1, Column: 1})
// The AST preserves var.ami_id, local.instance_type, and the merge() call
// as raw tokens - they are NOT evaluated

// We can manipulate blocks and attributes:
body := file.Body()
for _, block := range body.Blocks() {
    if block.Type() == "resource" {
        blockBody := block.Body()
        // Read existing attribute tokens
        attr := blockBody.GetAttribute("instance_type")
        // Modify or add attributes while preserving expression tokens
        blockBody.SetAttributeValue("monitoring", cty.True)
    }
}

// Serialize back to HCL - expressions are preserved!
output := file.Bytes()
```

### 2.4 Project Structure

```
kustomize-tf/
├── cmd/
│   └── kustomize-tf/
│       └── main.go              # Entry point
├── internal/
│   ├── cli/
│   │   ├── root.go              # Root command
│   │   ├── plan.go              # plan command
│   │   ├── apply.go             # apply command
│   │   ├── build.go             # build command (output merged config)
│   │   ├── destroy.go           # destroy command
│   │   └── init.go              # init command
│   ├── config/
│   │   ├── loader.go            # Load kustomize.tf
│   │   ├── parser.go            # Parse kustomize.tf content
│   │   └── validator.go         # Validate configuration
│   ├── hcl/
│   │   ├── parser.go            # Parse .tf files
│   │   ├── writer.go            # Write .tf files
│   │   └── utils.go             # HCL utilities
│   ├── merge/
│   │   ├── engine.go            # Core merge orchestration
│   │   ├── blocks.go            # Block merging logic
│   │   ├── attributes.go        # Attribute merging logic
│   │   ├── maps.go              # Map deep merge logic
│   │   ├── resources.go         # Resource-specific merge rules
│   │   ├── variables.go         # Variable block merging
│   │   ├── outputs.go           # Output block merging
│   │   ├── locals.go            # Locals block merging
│   │   ├── modules.go           # Module block merging
│   │   ├── providers.go         # Provider block merging
│   │   └── terraform.go         # Terraform block merging
│   ├── executor/
│   │   ├── terraform.go         # Execute terraform commands
│   │   └── process.go           # Process management
│   └── workspace/
│       ├── temp.go              # Temp directory management
│       └── cleanup.go           # Cleanup handlers
├── pkg/
│   └── version/
│       └── version.go           # Version info
├── test/
│   ├── fixtures/                # Test fixtures
│   │   ├── basic/
│   │   ├── deep-merge/
│   │   ├── modules/
│   │   └── complex/
│   ├── integration/             # Integration tests
│   └── e2e/                     # End-to-end tests
├── examples/
│   ├── simple/
│   ├── multi-env/
│   └── with-modules/
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

---

## 3. Merge Semantics Deep Dive

### 3.1 Block Type Reference

| Block Type | Identity Key | Merge Behavior |
|------------|--------------|----------------|
| `resource` | type + name | Deep merge attributes/nested blocks |
| `data` | type + name | Deep merge (same as resource) |
| `variable` | name | Replace default, merge validation |
| `output` | name | Replace value, merge other attrs |
| `locals` | (single block) | Deep merge all local values |
| `module` | name | Deep merge inputs |
| `provider` | name + alias | Deep merge attributes |
| `terraform` | (single block) | Special merge rules |

### 3.2 Detailed Merge Rules

#### 3.2.1 Resource and Data Blocks

**Identity:** `resource.aws_instance.web` or `data.aws_ami.latest`

**Rules:**
1. **Scalars** (strings, numbers, bools): Overlay replaces base
2. **Maps** (tags, labels, etc.): Deep merge - overlay keys override, base keys preserved
3. **Lists**: Overlay replaces entirely
4. **Nested blocks**: Overlay replaces all blocks of same type (matches Terraform override behavior)

> **Exception: `lifecycle` blocks** are merged at the attribute level, not replaced entirely. See [Section 3.3.3](#333-lifecycle-blocks) for details. This matches Terraform's override file behavior where lifecycle arguments are merged individually.

```hcl
# BASE
resource "aws_instance" "web" {
  ami           = "ami-base"
  instance_type = "t3.micro"

  tags = {
    Name = "base-web"
    Team = "platform"
  }

  ebs_block_device {
    device_name = "/dev/sda1"
    volume_size = 20
  }
}

# OVERLAY
resource "aws_instance" "web" {
  instance_type = "t3.large"     # REPLACES
  monitoring    = true           # ADDS

  tags = {
    Name        = "prod-web"     # OVERRIDES
    Environment = "prod"         # ADDS
    # Team preserved from base
  }

  ebs_block_device {             # REPLACES all ebs_block_device blocks
    device_name = "/dev/sda1"
    volume_size = 100
  }
}

# RESULT
resource "aws_instance" "web" {
  ami           = "ami-base"     # FROM BASE
  instance_type = "t3.large"     # FROM OVERLAY
  monitoring    = true           # FROM OVERLAY

  tags = {
    Name        = "prod-web"     # FROM OVERLAY
    Team        = "platform"     # FROM BASE
    Environment = "prod"         # FROM OVERLAY
  }

  ebs_block_device {
    device_name = "/dev/sda1"
    volume_size = 100            # FROM OVERLAY
  }
}
```

#### 3.2.2 Variable Blocks

**Identity:** `variable.instance_type`

**Rules:**
1. `type`: Overlay replaces (if specified)
2. `default`: Overlay replaces
3. `description`: Overlay replaces
4. `validation`: Overlay **replaces** all validation blocks (matches Terraform override behavior)
5. `sensitive`: Overlay replaces
6. `nullable`: Overlay replaces

> **Note on validation blocks:** Following Terraform's override file semantics, if an overlay defines any validation blocks, they replace ALL base validation blocks. This ensures predictable behavior consistent with Terraform. If you need to preserve base validations while adding new ones, you must redeclare them in the overlay.
>
> **Future Enhancement (Phase 2):** Strategic merge with `$delete` directive will allow more granular control - adding validations without redeclaring, or selectively removing base validations. See [Phase 4 backlog](#phase-4-future-enhancements-backlog).

```hcl
# BASE
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type"

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Must be a t2 or t3 instance type."
  }
}

# OVERLAY
variable "instance_type" {
  default = "t3.large"   # REPLACES default only

  validation {
    condition     = !can(regex("^t[23]\\.nano", var.instance_type))
    error_message = "Nano instances not allowed in production."
  }
}

# RESULT - overlay validation REPLACES base validation
variable "instance_type" {
  type        = string           # FROM BASE
  default     = "t3.large"       # FROM OVERLAY
  description = "EC2 instance type"  # FROM BASE

  # Only overlay validation remains (base validation replaced)
  validation {
    condition     = !can(regex("^t[23]\\.nano", var.instance_type))
    error_message = "Nano instances not allowed in production."
  }
}

# To keep both validations, redeclare in overlay:
# OVERLAY (preserving base validation)
variable "instance_type" {
  default = "t3.large"

  validation {
    condition     = can(regex("^t[23]\\.", var.instance_type))
    error_message = "Must be a t2 or t3 instance type."
  }
  validation {
    condition     = !can(regex("^t[23]\\.nano", var.instance_type))
    error_message = "Nano instances not allowed in production."
  }
}
```

#### 3.2.3 Output Blocks

**Identity:** `output.instance_id`

**Rules:**
1. `value`: Overlay replaces
2. `description`: Overlay replaces
3. `sensitive`: Overlay replaces
4. `depends_on`: Overlay replaces
5. `precondition`: Overlay replaces all

```hcl
# BASE
output "instance_id" {
  value       = aws_instance.web.id
  description = "The instance ID"
}

# OVERLAY
output "instance_id" {
  description = "Production instance ID"  # REPLACES
  # value preserved from base
}
```

#### 3.2.4 Locals Block

**Identity:** Single `locals` block (all locals blocks are merged first, then overlay applied)

**Rules:**
1. Each local value is treated independently
2. Overlay local values replace base local values of the same name
3. Base local values not in overlay are preserved

```hcl
# BASE
locals {
  environment = "dev"
  common_tags = {
    Project = "myapp"
    Team    = "platform"
  }
}

# OVERLAY
locals {
  environment = "prod"          # REPLACES
  region      = "us-east-1"     # ADDS
  common_tags = {               # REPLACES ENTIRELY (it's a local value, not a map attribute)
    Project     = "myapp"
    Team        = "platform"
    Environment = "prod"
  }
}

# RESULT
locals {
  environment = "prod"
  region      = "us-east-1"
  common_tags = {
    Project     = "myapp"
    Team        = "platform"
    Environment = "prod"
  }
}
```

> **Design Rationale: Why Locals Use Replacement Instead of Deep Merge**
>
> Unlike resource attributes (e.g., `tags`) which are known to be maps and can be safely deep-merged, local values have several characteristics that make deep merging problematic:
>
> 1. **Type ambiguity:** A local value can be any type (string, number, bool, list, map, object, or complex expression). The tool cannot reliably determine the type without evaluating expressions.
>
> 2. **Expression semantics:** Local values often contain expressions like `merge(var.a, var.b)` or complex transformations. Deep merging into such expressions would require expression analysis and rewriting, which is error-prone.
>
> 3. **Intentional overrides:** When an overlay specifies a local value, it typically intends to completely replace the base value to customize behavior for that environment.
>
> 4. **Consistency with Terraform:** Terraform's override files also replace local values entirely.
>
> **If you need to extend a base map in locals**, use Terraform's `merge()` function in the overlay:
> ```hcl
> # overlay/locals.tf - explicitly merge with base
> locals {
>   common_tags = merge(
>     { Project = "myapp", Team = "platform" },  # base values
>     { Environment = "prod" }                    # overlay additions
>   )
> }
> ```

#### 3.2.5 Module Blocks

**Identity:** `module.vpc`

**Rules:**
1. `source`: Overlay replaces (allows pointing to different module)
2. `version`: Overlay replaces
3. All input variables: Deep merge (maps deep merge, scalars replace)
4. `providers`: Overlay replaces
5. `depends_on`: Overlay replaces
6. `for_each`/`count`: Overlay replaces

```hcl
# BASE
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.0.0"

  name = "base-vpc"
  cidr = "10.0.0.0/16"

  tags = {
    Team = "platform"
  }
}

# OVERLAY
module "vpc" {
  name = "prod-vpc"           # REPLACES
  cidr = "10.1.0.0/16"        # REPLACES

  enable_nat_gateway = true   # ADDS

  tags = {
    Environment = "prod"      # MERGES
  }
}

# RESULT
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"  # FROM BASE
  version = "3.0.0"                          # FROM BASE

  name = "prod-vpc"           # FROM OVERLAY
  cidr = "10.1.0.0/16"        # FROM OVERLAY
  enable_nat_gateway = true   # FROM OVERLAY

  tags = {
    Team        = "platform"  # FROM BASE
    Environment = "prod"      # FROM OVERLAY
  }
}
```

#### 3.2.6 Provider Blocks

**Identity:** `provider.aws` or `provider.aws.west` (with alias)

**Rules:**
1. All attributes: Deep merge (maps deep merge, scalars replace)
2. Overlay can add new provider configurations
3. Alias is part of identity, not merged

```hcl
# BASE
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy = "terraform"
    }
  }
}

# OVERLAY
provider "aws" {
  region = "us-west-2"        # REPLACES

  default_tags {
    tags = {
      Environment = "prod"    # MERGES
    }
  }
}

provider "aws" {
  alias  = "west"             # NEW provider config
  region = "us-west-2"
}

# RESULT
provider "aws" {
  region = "us-west-2"        # FROM OVERLAY (replaced)

  default_tags {
    tags = {
      ManagedBy   = "terraform"  # FROM BASE (preserved)
      Environment = "prod"       # FROM OVERLAY (added)
    }
  }
}

provider "aws" {
  alias  = "west"             # NEW provider from overlay
  region = "us-west-2"
}
```

#### 3.2.7 Terraform Block

**Identity:** Single `terraform` block

**Rules:**
1. `required_version`: Overlay replaces
2. `required_providers`: Deep merge by provider name
3. `backend`: Overlay replaces entirely
4. `cloud`: Overlay replaces entirely
5. `experiments`: Overlay replaces

```hcl
# BASE
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0.0"
    }
  }

  backend "s3" {
    bucket = "base-state"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}

# OVERLAY
terraform {
  required_providers {
    aws = {
      version = ">= 5.0.0"    # REPLACES version for aws
    }
    random = {                 # ADDS new provider
      source  = "hashicorp/random"
      version = ">= 3.0.0"
    }
  }

  backend "s3" {               # REPLACES ENTIRE backend
    bucket = "prod-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

# RESULT
terraform {
  required_version = ">= 1.0.0"   # FROM BASE

  required_providers {
    aws = {
      source  = "hashicorp/aws"   # FROM BASE
      version = ">= 5.0.0"        # FROM OVERLAY
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0.0"
    }
  }

  backend "s3" {
    bucket = "prod-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### 3.3 Edge Cases and Special Handling

#### 3.3.1 Dynamic Blocks

**Approach:** Dynamic blocks are treated as nested blocks and follow the nested block replacement rule.

```hcl
# BASE
resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}

# OVERLAY with dynamic block
resource "aws_security_group" "web" {
  dynamic "ingress" {           # REPLACES all ingress (including dynamic)
    for_each = var.prod_ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

#### 3.3.2 Count and For-Each

**Approach:** These are meta-arguments, treated as regular attributes (overlay replaces).

```hcl
# BASE
resource "aws_instance" "web" {
  count = 1
  ami   = "ami-123"
}

# OVERLAY
resource "aws_instance" "web" {
  count = 3   # REPLACES
}

# OR: Remove count, add for_each
resource "aws_instance" "web" {
  for_each = toset(["a", "b", "c"])   # REPLACES count
}
```

> **Warning:** Changing from `count` to `for_each` or vice versa may require state migration. This tool does not handle that automatically.

#### 3.3.3 Lifecycle Blocks

**Approach:** Follow Terraform's override behavior - lifecycle contents are merged argument-by-argument.

```hcl
# BASE
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true
    ignore_changes        = [tags]
  }
}

# OVERLAY
resource "aws_instance" "web" {
  lifecycle {
    prevent_destroy = true    # ADDS
    # create_before_destroy and ignore_changes preserved
  }
}

# RESULT
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true   # FROM BASE
    ignore_changes        = [tags] # FROM BASE
    prevent_destroy       = true   # FROM OVERLAY
  }
}
```

#### 3.3.4 Provisioners and Connections

**Approach:** Follow Terraform's override behavior:
- If overlay has **any** provisioner block, ALL base provisioners are ignored (global replacement, not type-specific)
- Connection block in overlay completely overrides base

> **Clarification:** Provisioner replacement is global, not type-specific. If the overlay defines a `local-exec` provisioner, it replaces ALL base provisioners including `remote-exec`, `file`, etc. This matches Terraform's override file behavior.

```hcl
# BASE
resource "aws_instance" "web" {
  provisioner "remote-exec" {
    inline = ["echo base-remote"]
  }
  provisioner "local-exec" {
    command = "echo base-local"
  }
}

# OVERLAY - has ONE provisioner
resource "aws_instance" "web" {
  provisioner "remote-exec" {
    inline = ["echo overlay"]
  }
}

# RESULT - ALL base provisioners replaced, only overlay provisioner remains
resource "aws_instance" "web" {
  provisioner "remote-exec" {
    inline = ["echo overlay"]
  }
  # local-exec from base is GONE
}
```

#### 3.3.5 Depends-On

**Decision:** `depends_on` in overlays is NOT allowed (following Terraform's behavior with override files).

If an overlay specifies `depends_on`, the tool should emit an error:
```
Error: depends_on cannot be specified in overlays
  on overlays/prod/main.tf line 5:
  5:   depends_on = [aws_vpc.main]

The depends_on meta-argument cannot be used in overlay configurations.
Define dependencies in the base configuration instead.
```

### 3.4 Map Detection for Deep Merge

The critical question: **How do we know if an attribute is a map that should be deep merged vs. a scalar that should be replaced?**

**Approach: Heuristic-based detection**

1. **Known map attributes:** Maintain a list of known map attributes for common resources:
   - `tags` (all AWS resources)
   - `labels` (all GCP/K8s resources)
   - `annotations` (K8s resources)
   - `default_tags` (provider blocks)
   - `metadata` (various)
   - `environment` (Lambda, ECS)

2. **Structural detection:** If an attribute's value is an object literal `{ key = value }`, treat it as a map for merging purposes.

3. **Expression handling:** If the attribute value is an expression (e.g., `var.tags`, `merge(...)`, `local.tags`), we cannot merge it - the overlay value replaces entirely.

```go
// Pseudo-code for map detection
func isDeepMergeableMap(attrName string, value *hclwrite.Attribute) bool {
    // Check known map attributes
    if knownMapAttributes[attrName] {
        // Only deep merge if value is an object literal
        return isObjectLiteral(value)
    }
    return false
}

func isObjectLiteral(attr *hclwrite.Attribute) bool {
    tokens := attr.Expr().BuildTokens(nil)
    // Check if first meaningful (non-whitespace) token is '{'
    // This indicates an object literal vs. a reference or function call
    for _, tok := range tokens {
        switch tok.Type {
        // Skip whitespace and newlines
        case hclsyntax.TokenNewline, hclsyntax.TokenComment:
            continue
        // Object literal starts with '{'
        case hclsyntax.TokenOBrace:
            return true
        // Reference (var.x, local.x) or function call starts with identifier
        case hclsyntax.TokenIdent, hclsyntax.TokenDot:
            return false
        // Any other token means it's not a simple object literal
        default:
            return false
        }
    }
    return false
}
```

> **Note:** The hclwrite package's token types may vary. During implementation, verify the exact token type constants available in `hclsyntax`. The key insight is to skip non-semantic tokens (whitespace, comments) before checking the first meaningful token.

### 3.5 Merge Conflict Detection

Some merges are "conflicting" in the sense that they might not produce expected results:

1. **Type change:** Base has `count`, overlay has `for_each`
2. **Expression to literal:** Base has `tags = var.tags`, overlay has `tags = { Name = "foo" }`
3. **Incompatible nested blocks:** Base has static block, overlay has dynamic block of same type

**Approach:** Emit warnings (not errors) for potential conflicts:

```
Warning: Merge conflict detected in resource.aws_instance.web
  The 'tags' attribute in base uses an expression (var.common_tags)
  but overlay specifies a literal map. The overlay will replace entirely.
  Consider using merge() in the overlay to combine values:
    tags = merge(var.common_tags, { Environment = "prod" })
```

### 3.6 Lock File and State Management

#### 3.6.1 `.terraform.lock.hcl` Handling

The dependency lock file (`.terraform.lock.hcl`) requires special consideration since base and overlay may specify different provider versions.

**Strategy:**

1. **Overlay takes precedence:** If an overlay directory contains a `.terraform.lock.hcl` file, it is used exclusively
2. **Fall back to base:** If no overlay lock file exists, use the base lock file
3. **Validation:** When both exist, emit a warning if they specify conflicting provider versions

```
Warning: Lock file conflict detected
  Base specifies: hashicorp/aws = 4.67.0
  Overlay specifies: hashicorp/aws = 5.31.0

  Using overlay's .terraform.lock.hcl.
  Run 'kustomize-tf init -k overlays/prod -- -upgrade' to update dependencies.
```

**Recommendation:** Users should maintain lock files per overlay to ensure reproducible builds. The overlay's lock file represents the actual deployed state for that environment.

#### 3.6.2 Terraform State Management

**Critical Requirement:** Since kustomize-tf executes terraform in temporary directories that are deleted after each invocation, **all state must be managed via remote backends**.

**Supported Patterns:**

1. **Backend in overlay:** Define a unique backend per overlay
   ```hcl
   # overlays/prod/backend.tf
   terraform {
     backend "s3" {
       bucket = "mycompany-terraform-state"
       key    = "prod/myapp/terraform.tfstate"
       region = "us-east-1"
     }
   }
   ```

2. **Backend in base with overlay override:** Base defines backend structure, overlay customizes
   ```hcl
   # base/backend.tf
   terraform {
     backend "s3" {
       bucket = "mycompany-terraform-state"
       region = "us-east-1"
     }
   }

   # overlays/prod/backend.tf
   terraform {
     backend "s3" {
       key = "prod/myapp/terraform.tfstate"  # REPLACES entire backend
     }
   }
   ```

**Local State Warning:**

If no remote backend is configured, kustomize-tf should emit a warning:

```
Warning: No remote backend configured
  Local state files will not persist between kustomize-tf invocations.
  Configure a remote backend (S3, GCS, Azure, etc.) for production use.

  Alternatively, use --keep-temp to preserve the working directory.
```

#### 3.6.3 `.terraform` Directory Handling

The `.terraform` directory contains:
- Downloaded providers
- Downloaded modules
- Backend configuration cache

**Strategy:**

1. **Fresh init by default:** Each `kustomize-tf` invocation runs in a fresh temp directory
2. **Cache optimization (future):** Consider a shared provider cache via `TF_PLUGIN_CACHE_DIR`
3. **Module caching:** Leverage Terraform's native module caching

---

## 4. CLI Interface Design

### 4.1 Command Structure

```
kustomize-tf <command> [options] [-- terraform-args]

Commands:
  build      Output merged configuration to stdout or directory
  init       Initialize terraform in the merged configuration
  validate   Validate merged configuration without executing
  plan       Run terraform plan on merged configuration
  apply      Run terraform apply on merged configuration
  destroy    Run terraform destroy on merged configuration
  version    Print version information

Global Options:
  -k, --kustomization <path>   Path to overlay directory (required for most commands)
  -v, --verbose                Enable verbose output
  -q, --quiet                  Suppress non-error output
  --no-color                   Disable colored output
  --debug                      Enable debug logging
  --keep-temp                  Preserve temp directory after execution (useful for debugging)
  --temp-dir <path>            Use specified directory instead of creating temp (implies --keep-temp)

Terraform Passthrough:
  Arguments after -- are passed directly to terraform
```

### 4.2 Command Details

#### `kustomize-tf build`

```bash
# Output merged config to stdout
kustomize-tf build -k overlays/prod

# Output to a directory
kustomize-tf build -k overlays/prod -o /tmp/merged

# Include source comments (shows which file each block came from)
kustomize-tf build -k overlays/prod --with-sources
```

**Flags:**
- `-o, --output <path>`: Output directory (default: stdout)
- `--with-sources`: Add comments showing source of each block
- `--format`: Output format (hcl, json)

#### `kustomize-tf init`

```bash
# Initialize terraform
kustomize-tf init -k overlays/prod

# Pass args to terraform init
kustomize-tf init -k overlays/prod -- -upgrade -reconfigure
```

**Behavior:**
1. Merge configuration
2. Write to temp directory
3. Run `terraform init` with any passthrough args
4. Keep temp directory for subsequent commands (or cleanup if standalone)

#### `kustomize-tf plan`

```bash
# Plan changes
kustomize-tf plan -k overlays/prod

# With terraform options
kustomize-tf plan -k overlays/prod -- -out=plan.tfplan -var="env=prod"

# Keep temp directory for inspection or to reuse plan file
kustomize-tf plan -k overlays/prod --keep-temp -- -out=plan.tfplan
# Output: Temp directory preserved: /tmp/kustomize-tf-abc123
# The plan file will be at /tmp/kustomize-tf-abc123/plan.tfplan

# Use a specific directory (useful for CI/CD or plan+apply workflows)
kustomize-tf plan -k overlays/prod --temp-dir ./terraform-work -- -out=plan.tfplan
kustomize-tf apply -k overlays/prod --temp-dir ./terraform-work -- plan.tfplan
```

#### `kustomize-tf apply`

```bash
# Apply changes (with confirmation)
kustomize-tf apply -k overlays/prod

# Auto-approve
kustomize-tf apply -k overlays/prod -- -auto-approve

# From saved plan
kustomize-tf apply -k overlays/prod -- plan.tfplan
```

#### `kustomize-tf destroy`

```bash
# Destroy resources
kustomize-tf destroy -k overlays/prod

# Auto-approve
kustomize-tf destroy -k overlays/prod -- -auto-approve
```

#### `kustomize-tf validate`

```bash
# Validate merged configuration
kustomize-tf validate -k overlays/prod
```

**Behavior:**
1. Merge configuration
2. Run `terraform validate`
3. Report merge warnings and terraform validation results

### 4.3 kustomize.tf File Format

```hcl
# overlays/prod/kustomize.tf

kustomize {
  # Base configuration to merge with
  # Can be relative path from overlay directory
  base = "../../../base"

  # Alternative: multiple bases (merged in order, first = lowest priority)
  # bases = ["../../base", "../../common"]

  # Additional resources to include (optional)
  resources = [
    "additional.tf"
  ]

  # Name for this overlay (optional, used in logging)
  name = "production"
}
```

**Configuration Rules:**

1. **`base` vs `bases`:** These are mutually exclusive. If both are specified, emit an error:
   ```
   Error: Invalid kustomize.tf configuration
     on overlays/prod/kustomize.tf

     Cannot specify both 'base' and 'bases'. Use 'base' for a single base
     directory, or 'bases' for multiple bases merged in order.
   ```

2. **Multiple bases merge order:** When using `bases`, directories are merged left-to-right. The first base is lowest priority, subsequent bases override earlier ones, and the overlay has highest priority:
   ```hcl
   bases = ["../../base", "../../common", "../../security"]
   # Merge order: base → common → security → overlay (current dir)
   ```

3. **Required field:** Either `base` or `bases` must be specified (exactly one).

### 4.4 Error Handling and User Feedback

**Error Categories:**

1. **Configuration Errors** (exit code 1)
   ```
   Error: kustomize.tf not found
     Cannot find kustomize.tf in overlays/prod

     Expected location: overlays/prod/kustomize.tf
   ```

2. **Parse Errors** (exit code 1)
   ```
   Error: Failed to parse HCL
     in base/main.tf:15:3

     Unexpected token: expected '=' but got '{'
   ```

3. **Merge Errors** (exit code 1)
   ```
   Error: Conflicting block definitions
     resource.aws_instance.web is defined in multiple overlay files:
       - overlays/prod/main.tf:10
       - overlays/prod/instances.tf:5

     Each block type+name must appear at most once per directory.
   ```

4. **Merge Warnings** (continue with warning)
   ```
   Warning: Potential merge issue
     resource.aws_instance.web in overlays/prod/main.tf:10

     The 'count' meta-argument is being replaced with 'for_each'.
     This will require terraform state migration.
   ```

5. **Terraform Errors** (exit code from terraform)
   ```
   Error: Terraform execution failed
     terraform plan exited with code 1

     [terraform output follows]
   ```

---

## 5. Implementation Phases

### Phase 1: MVP (Core Functionality)
**Duration Estimate:** 3-4 weeks

#### Milestone 1.1: Project Setup & CLI Framework
- [ ] Initialize Go module with dependencies
- [ ] Set up cobra CLI framework
- [ ] Implement `version` command
- [ ] Set up basic logging infrastructure
- [ ] Create project structure

**Deliverable:** `kustomize-tf version` works

#### Milestone 1.2: Configuration Loading
- [ ] Implement kustomize.tf parser
- [ ] Implement path resolution (base/overlay)
- [ ] Implement .tf file discovery in directories
- [ ] Add validation for configuration

**Deliverable:** Can load and validate overlay structure

#### Milestone 1.3: HCL Parsing
- [ ] Implement HCL file parser using hclwrite
- [ ] Parse all .tf files in a directory
- [ ] Extract block identities (type + labels)
- [ ] Preserve expression tokens

**Deliverable:** Can parse base and overlay .tf files into AST

#### Milestone 1.4: Basic Merge Engine
- [ ] Implement block matching by identity
- [ ] Implement scalar attribute replacement
- [ ] Implement nested block replacement
- [ ] Handle new blocks from overlay

**Deliverable:** Basic merge works for simple cases

#### Milestone 1.5: Map Deep Merge
- [ ] Implement map detection heuristics
- [ ] Implement deep merge for known map attributes (tags)
- [ ] Add tests for map merge scenarios

**Deliverable:** Tags and similar maps merge correctly

#### Milestone 1.6: Build Command
- [ ] Implement `build` command
- [ ] Output merged HCL to stdout
- [ ] Implement `-o` flag for file output
- [ ] Add formatting (proper HCL output)

**Deliverable:** `kustomize-tf build -k overlays/prod` works

#### Milestone 1.7: Terraform Execution
- [ ] Implement temp directory management
- [ ] Implement terraform command execution
- [ ] Implement output streaming
- [ ] Implement cleanup on exit/interrupt

**Deliverable:** Can execute terraform in temp directory

#### Milestone 1.8: Core Commands
- [ ] Implement `init` command
- [ ] Implement `plan` command
- [ ] Implement `apply` command
- [ ] Implement `destroy` command
- [ ] Implement `validate` command
- [ ] Handle terraform passthrough arguments

**Deliverable:** All core commands work

### Phase 2: Block-Specific Merge Rules
**Duration Estimate:** 2-3 weeks

#### Milestone 2.1: Variable Block Merge
- [ ] Implement variable-specific merge rules
- [ ] Preserve type if not in overlay
- [ ] Preserve description if not in overlay
- [ ] Handle validation block replacement

#### Milestone 2.2: Output Block Merge
- [ ] Implement output-specific merge rules
- [ ] Handle precondition block replacement

#### Milestone 2.3: Module Block Merge
- [ ] Implement module-specific merge rules
- [ ] Deep merge module inputs
- [ ] Handle source/version replacement

#### Milestone 2.4: Provider Block Merge
- [ ] Implement provider-specific merge rules
- [ ] Handle alias as part of identity
- [ ] Deep merge provider attributes

#### Milestone 2.5: Terraform Block Merge
- [ ] Implement terraform block merge
- [ ] Deep merge required_providers
- [ ] Handle backend replacement
- [ ] Handle cloud block

#### Milestone 2.6: Lifecycle Block Special Handling
- [ ] Implement lifecycle argument-by-argument merge
- [ ] Follow Terraform override behavior

#### Milestone 2.7: Provisioner Handling
- [ ] Implement provisioner replacement rule
- [ ] Emit warnings for provisioner merge

### Phase 3: Polish & Edge Cases
**Duration Estimate:** 2-3 weeks

#### Milestone 3.1: Error Handling
- [ ] Implement comprehensive error messages
- [ ] Add source location in errors
- [ ] Implement warning system
- [ ] Add suggestions in error messages

#### Milestone 3.2: Conflict Detection
- [ ] Detect count/for_each conflicts
- [ ] Detect expression/literal conflicts
- [ ] Emit helpful warnings

#### Milestone 3.3: Multiple Bases
- [ ] Implement multiple base support
- [ ] Define merge order
- [ ] Handle conflicts between bases

#### Milestone 3.4: State Management
- [ ] Investigate state file handling
- [ ] Handle .terraform directory
- [ ] Handle .terraform.lock.hcl

#### Milestone 3.5: Documentation
- [ ] Write README
- [ ] Create usage examples
- [ ] Document merge semantics
- [ ] Add inline help text

### Phase 4: Future Enhancements (Backlog)

- **Strategic merge patches** (Kustomize-style granular control)
  - `$patch: delete` - Remove specific blocks/attributes from base
  - `$patch: replace` - Explicit full replacement
  - `$patch: merge` - Force additive merge for nested blocks (e.g., validations)
  - Example:
    ```hcl
    # Remove a specific validation from base
    variable "instance_type" {
      validation {
        $patch = "delete"
        condition = can(regex("^t[23]\\.", var.instance_type))  # identifies which to delete
      }
    }
    ```
- Transformers (auto-add tags to all resources)
- Generators (generate config from external sources)
- Components (reusable overlay fragments)
- Remote bases (git URLs)
- Validation hooks
- IDE extensions

---

## 6. Testing Strategy

### 6.1 Unit Tests

**Coverage Target:** 80%+ for merge logic

```go
// Example: Testing scalar replacement
func TestMergeScalarAttribute(t *testing.T) {
    base := `resource "aws_instance" "web" { instance_type = "t3.micro" }`
    overlay := `resource "aws_instance" "web" { instance_type = "t3.large" }`

    result, err := merge(parse(base), parse(overlay))
    require.NoError(t, err)

    assert.Contains(t, result, `instance_type = "t3.large"`)
}

// Example: Testing map deep merge
func TestMergeMapDeepMerge(t *testing.T) {
    base := `resource "aws_instance" "web" { tags = { Team = "platform" } }`
    overlay := `resource "aws_instance" "web" { tags = { Env = "prod" } }`

    result, err := merge(parse(base), parse(overlay))
    require.NoError(t, err)

    assert.Contains(t, result, `Team = "platform"`)
    assert.Contains(t, result, `Env = "prod"`)
}
```

**Test Categories:**
1. Block matching tests
2. Scalar attribute replacement tests
3. Map deep merge tests
4. List replacement tests
5. Nested block replacement tests
6. Edge case tests (expressions, dynamic blocks, etc.)

### 6.2 Integration Tests

**Approach:** Use fixture directories with expected outputs

```
test/fixtures/
├── basic-merge/
│   ├── base/
│   │   └── main.tf
│   ├── overlay/
│   │   ├── kustomize.tf
│   │   └── main.tf
│   └── expected/
│       └── main.tf
├── deep-merge-tags/
│   └── ...
├── module-override/
│   └── ...
└── terraform-block/
    └── ...
```

```go
func TestFixtures(t *testing.T) {
    fixtures, _ := filepath.Glob("test/fixtures/*/")
    for _, fixture := range fixtures {
        t.Run(filepath.Base(fixture), func(t *testing.T) {
            result := runBuild(filepath.Join(fixture, "overlay"))
            expected := readFile(filepath.Join(fixture, "expected", "main.tf"))
            assert.Equal(t, normalize(expected), normalize(result))
        })
    }
}
```

### 6.3 End-to-End Tests

**Approach:** Actually run terraform against merged configs

```go
func TestE2E_PlanApply(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping E2E test")
    }

    // Use LocalStack or terraform with null_resource
    overlay := "test/e2e/simple-overlay"

    // Run init
    err := runCommand("init", "-k", overlay)
    require.NoError(t, err)

    // Run plan
    err = runCommand("plan", "-k", overlay)
    require.NoError(t, err)

    // Verify plan output
    // ...
}
```

### 6.4 Test Scenarios

| Scenario | Base | Overlay | Expected |
|----------|------|---------|----------|
| Add new resource | 1 resource | 1 new resource | 2 resources |
| Modify resource | instance_type=micro | instance_type=large | large |
| Deep merge tags | {Team=A} | {Env=prod} | {Team=A, Env=prod} |
| Override tag | {Name=base} | {Name=prod} | {Name=prod} |
| Add attribute | ami only | +monitoring | ami + monitoring |
| Replace list | ingress[A] | ingress[B,C] | ingress[B,C] |
| New provider | aws | +aws.west | aws + aws.west |
| Override backend | s3:base | s3:prod | s3:prod |
| Module inputs | {name=base} | {name=prod, extra=true} | merged |

### 6.5 Golden File Tests

For complex merges, use golden files:

```go
func TestComplexMerge(t *testing.T) {
    result := runBuild("test/fixtures/complex")

    golden := "test/fixtures/complex/golden.tf"
    if *update {
        os.WriteFile(golden, []byte(result), 0644)
    }

    expected, _ := os.ReadFile(golden)
    assert.Equal(t, string(expected), result)
}
```

---

## 7. Open Questions & Risks

### 7.1 Open Questions Requiring Decision

#### Q1: Should we support JSON format (.tf.json)?

**Options:**
- A) Yes, parse and merge JSON HCL files
- B) No, only support native HCL
- C) Support reading JSON but output only HCL

**Recommendation:** B for MVP, add JSON support later. The hclwrite package focuses on HCL syntax.

#### Q2: How to handle file-level conflicts?

If overlay has `main.tf` and base has `main.tf`, they're both merged. But what if overlay has `instances.tf` defining the same resource as base's `main.tf`?

**Recommendation:** Merge works at block level, not file level. Blocks are merged by identity regardless of filename. Duplicate definitions in overlay (same identity in multiple files) should error.

#### Q3: Should we preserve comments?

hclwrite can preserve comments, but merging comments is complex.

**Options:**
- A) Preserve all comments from both
- B) Preserve only comments attached to surviving blocks
- C) Strip all comments

**Recommendation:** B - preserve comments that are attached to blocks, using hclwrite's comment preservation.

#### Q4: What about Terraform workspace support?

Should the tool be workspace-aware?

**Recommendation:** Pass through. The tool should not manage workspaces directly; workspace selection can be done via terraform args (`-workspace=` or `TF_WORKSPACE`).

#### Q5: Minimum Terraform version?

**Recommendation:** Support Terraform >= 1.0.0. This covers all modern Terraform features and has stable HCL syntax.

### 7.2 Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| hclwrite doesn't handle some HCL edge cases | High | Medium | Extensive testing, fallback to raw token manipulation |
| Expression preservation breaks during merge | High | Low | Use hclwrite's token-level API, never evaluate expressions |
| Users expect deeper merge than we provide | Medium | Medium | Clear documentation, warnings for non-mergeable scenarios |
| Performance with large configs | Low | Low | Lazy loading, parallel file processing |
| State file complications | High | Medium | Clear documentation that state is per-overlay, don't touch state files |
| Provider-specific map attributes | Medium | High | Start with common attributes (tags), add configuration for custom maps |

### 7.3 Future Considerations

1. **Remote Bases:** Support `base = "git::https://github.com/org/repo//path?ref=v1.0.0"`
2. **Schema-Aware Merging:** Load provider schemas for type-aware merging
3. **Dry-Run Mode:** Show what would be merged without executing
4. **Diff View:** Show unified diff of base vs merged
5. **Watch Mode:** Rebuild on file changes
6. **Import from Terragrunt:** Migration path for Terragrunt users

---

## Appendix A: Research Summary

### Existing Tools Comparison

| Tool | Approach | Strengths | Weaknesses |
|------|----------|-----------|------------|
| [Terragrunt](https://terragrunt.gruntwork.io/) | Wrapper, DRY configs | Mature, widely used | Different paradigm (not overlay-based) |
| [Atmos](https://atmos.tools/) | Components + Stacks | Powerful inheritance | Complex, steep learning curve |
| [Terramate](https://terramate.io/) | Stacks + Code Gen | Modern, change detection | Different focus (orchestration) |
| Terraform Override Files | Native | Built-in | Limited to same directory |

### Library References

- [hashicorp/hcl v2](https://github.com/hashicorp/hcl) - Official HCL library
- [hclwrite package](https://pkg.go.dev/github.com/hashicorp/hcl/v2/hclwrite) - AST manipulation
- [hclmerge](https://github.com/lonegunmanb/hclmerge) - Existing HCL merge tool (follows override semantics)
- [terraform-config-inspect](https://pkg.go.dev/github.com/hashicorp/terraform-config-inspect/tfconfig) - High-level config parsing

---

## Appendix B: Example Implementation Snippets

> **API Note:** The code examples below use hclwrite package methods. During implementation, verify the exact API signatures as they may differ slightly from these examples. Key methods to verify:
> - `Body.SetAttributeRaw(name string, tokens hclwrite.Tokens)` - may need `SetAttributeRaw(name, tokens)` or similar
> - `Body.GetAttribute(name string)` returns `*Attribute` or `nil`
> - `Attribute.Expr().BuildTokens(nil)` returns token sequence
>
> Refer to [hclwrite package documentation](https://pkg.go.dev/github.com/hashicorp/hcl/v2/hclwrite) for authoritative API details.

### B.1 Block Matching

```go
func matchBlocks(base, overlay *hclwrite.Body) map[string]*blockPair {
    pairs := make(map[string]*blockPair)

    // Index base blocks
    for _, block := range base.Blocks() {
        id := blockIdentity(block)
        pairs[id] = &blockPair{base: block}
    }

    // Match overlay blocks
    for _, block := range overlay.Blocks() {
        id := blockIdentity(block)
        if pair, ok := pairs[id]; ok {
            pair.overlay = block
        } else {
            pairs[id] = &blockPair{overlay: block}
        }
    }

    return pairs
}

func blockIdentity(block *hclwrite.Block) string {
    labels := block.Labels()
    return block.Type() + "." + strings.Join(labels, ".")
}
```

### B.2 Attribute Merge

```go
// mergeAttributes merges attributes from overlay into a result body.
// It handles three cases:
// 1. Attribute only in base -> copy to result unchanged
// 2. Attribute only in overlay -> copy to result
// 3. Attribute in both -> merge (deep merge for maps, replace for scalars)
func mergeAttributes(base, overlay *hclwrite.Body, knownMaps map[string]bool) *hclwrite.Body {
    // Create a new result body to avoid modifying inputs
    result := hclwrite.NewEmptyFile().Body()

    // Track which attributes we've processed
    processed := make(map[string]bool)

    // First, handle all base attributes
    for name, baseAttr := range base.Attributes() {
        overlayAttr := overlay.GetAttribute(name)

        if overlayAttr == nil {
            // Case 1: Only in base - copy unchanged
            result.SetAttributeRaw(name, baseAttr.Expr().BuildTokens(nil))
        } else {
            // Case 3: In both - merge or replace
            if knownMaps[name] && isObjectLiteral(baseAttr) && isObjectLiteral(overlayAttr) {
                // Deep merge maps
                merged := deepMergeMaps(baseAttr, overlayAttr)
                result.SetAttributeRaw(name, merged)
            } else {
                // Replace with overlay value
                result.SetAttributeRaw(name, overlayAttr.Expr().BuildTokens(nil))
            }
        }
        processed[name] = true
    }

    // Then, add any overlay-only attributes
    for name, overlayAttr := range overlay.Attributes() {
        if !processed[name] {
            // Case 2: Only in overlay - copy to result
            result.SetAttributeRaw(name, overlayAttr.Expr().BuildTokens(nil))
        }
    }

    return result
}
```

### B.3 Temp Directory Execution

```go
// ExecutorOptions configures the terraform executor behavior
type ExecutorOptions struct {
    KeepTemp bool   // If true, don't delete temp directory after execution
    TempDir  string // If set, use this directory instead of creating a new one
}

func (e *Executor) Run(cmd string, args []string, opts ExecutorOptions) error {
    var tmpDir string
    var err error

    // Create or use existing temp directory
    if opts.TempDir != "" {
        tmpDir = opts.TempDir
        // Ensure directory exists
        if err := os.MkdirAll(tmpDir, 0700); err != nil {
            return fmt.Errorf("failed to create temp directory %s: %w", tmpDir, err)
        }
    } else {
        tmpDir, err = os.MkdirTemp("", "kustomize-tf-*")
        if err != nil {
            return fmt.Errorf("failed to create temp directory: %w", err)
        }
    }

    // Schedule cleanup unless --keep-temp is set
    if !opts.KeepTemp {
        defer func() {
            if removeErr := os.RemoveAll(tmpDir); removeErr != nil {
                fmt.Fprintf(os.Stderr, "Warning: failed to cleanup temp directory %s: %v\n", tmpDir, removeErr)
            }
        }()
    } else {
        fmt.Fprintf(os.Stderr, "Temp directory preserved: %s\n", tmpDir)
    }

    // Write merged config files with restrictive permissions
    for filename, file := range e.merged {
        path := filepath.Join(tmpDir, filename)

        // Create parent directories if needed (for nested file structures)
        dir := filepath.Dir(path)
        if err := os.MkdirAll(dir, 0700); err != nil {
            return fmt.Errorf("failed to create directory %s: %w", dir, err)
        }

        // Write with restrictive permissions (0600) since configs may contain sensitive data
        if err := os.WriteFile(path, file.Bytes(), 0600); err != nil {
            return fmt.Errorf("failed to write %s: %w", path, err)
        }
    }

    // Execute terraform
    tfCmd := exec.Command("terraform", append([]string{cmd}, args...)...)
    tfCmd.Dir = tmpDir
    tfCmd.Stdout = os.Stdout
    tfCmd.Stderr = os.Stderr
    tfCmd.Stdin = os.Stdin

    if err := tfCmd.Run(); err != nil {
        return fmt.Errorf("terraform %s failed: %w", cmd, err)
    }

    return nil
}
```

---

*This plan was created on December 24, 2025*
