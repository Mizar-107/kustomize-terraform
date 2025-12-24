# Project: Plan a Kustomize-like Tool for Terraform

## Context and Vision

I want to build a CLI tool that brings Kustomize's elegant base + overlay pattern to Terraform. Currently, managing multiple environments (dev/staging/prod) with Terraform is not as elegant as managing Kubernetes manifests with Kustomize. This tool will allow teams to:

- Define a **base** Terraform configuration (reusable foundation)
- Create **overlays** that modify the base for specific environments/tenants
- Avoid code duplication while maintaining flexibility
- Execute terraform commands against merged configurations

## Key Decisions Already Made

### 1. Core Merge Semantics
When merging base + overlay HCL configurations:

- **Resources** are matched by `type + name` (e.g., `aws_instance.web`)
- **Scalar values**: Overlay REPLACES base
- **Maps/Objects** (tags, labels): DEEP MERGE (overlay adds/overrides keys)
- **Lists/Arrays**: Overlay REPLACES entirely
- **Overlay always takes priority** over base

Example:
```hcl
# base/main.tf
resource "aws_instance" "web" {
  instance_type = "t3.micro"
  ami           = "ami-12345"
  tags = {
    Name = "web-server"
    Team = "platform"
  }
}

# overlays/prod/main.tf
resource "aws_instance" "web" {
  instance_type = "t3.large"    # REPLACES
  monitoring    = true          # ADDS
  tags = {
    Environment = "prod"        # MERGES into tags
    Name        = "prod-web"    # OVERRIDES Name
  }
}

# Result: instance_type=t3.large, monitoring=true, 
#         tags={Name="prod-web", Team="platform", Environment="prod"}
```

### 2. Format Choices
- **Metadata file format**: HCL (not YAML) - should be `kustomize.tf` in each overlay
- **User experience**: Tool should execute terraform directly (not just generate files)
- **Execution**: Similar to how Kustomize works with kubectl - merge and execute seamlessly

### 3. State Management Approach
- Each overlay maintains its own Terraform state
- Separate state per overlay (they're managing potentially different resources)
- User configures backend in overlay or inherits from base

### 4. Initial Scope
**Phase 1 (MVP):**
- Base + overlay pattern
- Resource block merging with defined semantics
- Variables, outputs, locals merging
- Module support (add modules, override module inputs)
- Execute terraform commands (init, plan, apply, destroy)

**Phase 2 (Future):**
- Strategic merge patches
- Resource deletion from overlays
- Transformers (auto-add tags to all resources)
- Advanced validation

## Your Task: Create a Comprehensive Implementation Plan

**Think HARD about this. I need you to:**

1. **Design the complete architecture**
   - What components/modules are needed?
   - How should they interact?
   - What are the data structures?
   - What's the execution flow from CLI command to terraform execution?

2. **Think through the technical challenges**
   - How do you parse HCL properly?
   - How do you implement the merge logic correctly?
   - How do you handle Terraform's complexities (expressions, functions, interpolations)?
   - How do you preserve formatting and comments?
   - What are the edge cases?

3. **Define the implementation approach**
   - What language/tools should be used and why?
   - What libraries are essential?
   - What's the project structure?
   - How should testing be approached?

4. **Identify critical decisions that need to be made**
   - What ambiguities exist in the merge semantics?
   - What about dynamic blocks, count/for_each, lifecycle blocks?
   - How to handle provider configuration merging?
   - How to handle terraform.required_providers merging?
   - What about data sources?
   - What about provisioners?

5. **Plan the CLI interface**
   - What commands should exist?
   - What flags/options are needed?
   - How should it integrate with existing terraform workflows?
   - Error handling and user feedback strategy?

6. **Create a phased development roadmap**
   - Break down into concrete milestones
   - What's the absolute minimum MVP?
   - What order should features be built?
   - What can be deferred to later phases?

7. **Define success criteria**
   - How do you know each phase is complete?
   - What testing validates correctness?
   - What examples demonstrate it working?

## Directory Structure Context

Assume users will structure projects like:
```
project/
├── base/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── overlays/
    ├── dev/
    │   ├── kustomize.tf      # Points to base
    │   └── main.tf           # Overlay modifications
    ├── staging/
    │   └── ...
    └── prod/
        └── ...
```

## Expected CLI Usage
```bash
# Apply an overlay
kustomize-tf apply -k overlays/prod

# Plan an overlay
kustomize-tf plan -k overlays/dev

# See merged config (for debugging)
kustomize-tf build -k overlays/prod
```

## Critical Questions You MUST Address

1. **HCL Parsing**: How do you handle HCL's complexity (expressions, functions, references)?
2. **AST Manipulation**: Do you work at AST level or higher abstraction?
3. **Merge Conflicts**: How do you detect and report incompatible merges?
4. **Module Merging**: How exactly does module input overriding work?
5. **Dynamic Blocks**: How do you merge dynamic blocks in resources?
6. **Count/For-Each**: What happens when base has count and overlay doesn't (or vice versa)?
7. **Data Sources**: Should data blocks be mergeable like resources?
8. **Dependencies**: How do you handle resource dependencies across base/overlay?
9. **Validation**: How do you validate the merged config makes sense?
10. **Terraform Version Compatibility**: What versions should be supported?

## What I Need From You

**Create a detailed technical plan that includes:**

1. **Architecture Document**
   - Component diagram
   - Data flow
   - Key algorithms (especially merge logic)

2. **Technical Specifications**
   - Language choice with justification
   - Essential libraries
   - Project structure
   - File organization

3. **Implementation Phases**
   - Milestone breakdown
   - Feature dependencies
   - What to build first, second, third

4. **Merge Semantics Deep Dive**
   - Detailed rules for every HCL element type
   - Edge case handling
   - Examples for complex scenarios

5. **Testing Strategy**
   - Unit test approach
   - Integration test approach
   - Example test scenarios

6. **Open Questions Document**
   - What design decisions still need to be made?
   - What ambiguities exist?
   - What tradeoffs need discussion?

## Instructions

- **THINK HARDEST** - This is a complex project with many nuances
- **ASK QUESTIONS** - If something is ambiguous or you need clarification, ask me
- **BE SPECIFIC** - Don't just say "parse HCL", explain HOW with library choices and approach
- **CONSIDER EDGE CASES** - Think about what could go wrong
- **RESEARCH** - Look into existing tools (Terragrunt, Atmos, Terramate) and HCL libraries
- **PROPOSE ALTERNATIVES** - If you see a better approach than what I described, suggest it
- **IDENTIFY RISKS** - What could make this project fail or be difficult?

This plan will be used to actually implement the tool, so it needs to be thorough, well-thought-out, and actionable.

**Now create the comprehensive plan. Think deeply about every aspect. Ask any questions you need answered before finalizing the plan.**
