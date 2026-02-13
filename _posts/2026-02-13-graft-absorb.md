---
title: "How I Built Graft Absorb: Turning Terraform Drift into Code"
date: 2026-02-13 08:00:00 +0800
categories: [Tools]
tags: [graft, terraform, drift, configuration]
---

In the [last post](/posts/introducing-graft/), I introduced Graft—a tool for patching Terraform modules without forking them. I mentioned it was middleware for something bigger. This is that something.

## The Problem

Terraform modules should be black boxes. We manage them through input variables, and ideally, we never need to know what's inside. But when drift happens—someone changes a resource in the portal, an external process modifies a tag, a compliance tool enforces a setting—`terraform plan` exposes the internals. Suddenly you're staring at resource-level changes deep inside a module you didn't write.

The typical response is painful: read through the plan output, find every difference, trace each change back to its source, and manually update configuration files to match the actual state. Even worse, if the module doesn't expose the right variables, there's simply no way to update the configuration to eliminate the drift.

The common workaround is `lifecycle { ignore_changes }`. But ignoring changes is not the same as managing them. You're telling Terraform to look the other way—which means your code no longer reflects reality. I think recording the actual desired state in code is always better than ignoring changes blindly.

## The Idea

What if we could automate this? Read a `terraform plan`, extract the drift, and generate the code changes needed to make the configuration match reality.

In the ideal scenario, such a tool would trace drift back to module input variables and update them directly. But I found this impractical—the mapping between inputs and resource attributes isn't always straightforward. A module might derive a resource's `tags` from a combination of multiple variables, local values, and conditional logic. Reverse-engineering that reliably is a dead end.

But Graft already solves the hard part. It can apply changes directly to module resources, bypassing module inputs entirely. So instead of trying to update variables, I can generate a Graft manifest that patches the drifted resources in place.

## `graft absorb`

I created a new command—`graft absorb`—that takes a Terraform plan file as input, analyzes every detected drift, and generates a Graft manifest describing the changes needed to bring configuration in line with actual state.

```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
graft absorb plan.json
```

That's it. The generated manifest is a standard `.graft.hcl` file. Review it, run `graft build`, and your next `terraform plan` should show zero changes.

Multi-layer modules work out of the box. Graft already knows how to navigate nested module hierarchies and apply patches at the correct level. And since the manifest format is just Terraform HCL, it's expressive enough to handle complex changes—attribute overrides, block additions, removals, all of it.

## The Hard Part: Indexed Resources

The simple resources with no indexing are easy. The real challenge is `count` and `for_each` resources.

Consider a resource with `count = 5`, where only 2 instances have drifted. I don't want to generate a manifest that overrides all 5 instances. It should target only the drifted ones—and still work correctly when the module updates and the instance count changes.

### Attribute Drift

For attributes, I settled on a `lookup()` pattern:

```hcl
# count-indexed resources
tags = lookup({
  0 = { environment = "production", owner = "drifttest", project = "graft" }
  1 = { environment = "staging",    owner = "drifttest", project = "graft" }
}, count.index, graft.source)

# for_each-indexed resources
location = lookup({
  "api" = "westus"
  "web" = "centralus"
}, each.key, graft.source)
```

The map contains only the drifted instances. `lookup` checks whether the current `count.index` or `each.key` has an entry. If it does, the new value is used. If not, `graft.source` kicks in—referencing the original expression from the module source, leaving un-drifted instances completely untouched.

This is resilient to module updates. If the upstream module adds new instances, they'll fall through to `graft.source` and behave exactly as the module author intended.

### Block Drift

Block-type attributes (like `security_rule` or `os_disk`) need a different approach. You can't use `graft.source` as a fallback for blocks because the original source has static block definitions, not list expressions that `dynamic` blocks can iterate over.

Instead, `graft absorb` generates `dynamic` blocks with `lookup()` in the `for_each`:

```hcl
resource "azurerm_network_security_group" "nsg" {
  _graft {
    remove = ["security_rule"]  # Remove original static blocks
  }

  dynamic "security_rule" {
    for_each = lookup({
      0 = [
        { name = "allow-ssh",   priority = 100, direction = "Inbound", ... },
        { name = "allow-https", priority = 200, direction = "Inbound", ... }
      ]
      1 = [
        { name = "allow-http",  priority = 100, direction = "Inbound", ... },
        { name = "deny-all",    priority = 4096, direction = "Inbound", ... }
      ]
    }, count.index, [])
    content {
      name      = security_rule.value.name
      priority  = security_rule.value.priority
      direction = security_rule.value.direction
      # ...
    }
  }
}
```

Each key in the lookup map is an instance index (or `each.key` for `for_each` resources), and the value is a list of block objects for that instance. The `content` block uses `security_rule.value.attr` to reference each attribute.

The fallback here is `[]`—an empty list—not `graft.source`. This means instances without an entry in the lookup map produce no dynamic block iterations. Combined with `_graft { remove = ["security_rule"] }`, which strips the original static blocks, the dynamic blocks become the sole source of truth.

For single nested blocks like `os_disk`, the same pattern applies—the lookup value is just a single-element list:

```hcl
dynamic "os_disk" {
  for_each = lookup({
    "web" = [{ caching = "ReadWrite", disk_size_gb = 50, storage_account_type = "Premium_LRS" }]
    "api" = [{ caching = "ReadOnly",  disk_size_gb = 64, storage_account_type = "StandardSSD_LRS" }]
  }, each.key, [])
  content {
    caching              = os_disk.value.caching
    disk_size_gb         = os_disk.value.disk_size_gb
    storage_account_type = os_disk.value.storage_account_type
  }
}
```

## Try It

```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
graft absorb plan.json        # Generate the manifest
# Review absorb.graft.hcl
graft build                   # Apply patches
terraform plan                # Should show zero changes
```

The full code and more examples are at [github.com/ms-henglu/graft](https://github.com/ms-henglu/graft). For a step-by-step walkthrough with a real public module, check the [absorb guide](https://github.com/ms-henglu/graft/blob/main/docs/absorb-guide.md).

If you hit edge cases or have ideas for improving the generated output, [open an issue](https://github.com/ms-henglu/graft/issues). 
